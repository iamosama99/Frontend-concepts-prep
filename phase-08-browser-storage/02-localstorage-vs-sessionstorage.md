# LocalStorage vs. SessionStorage

## Quick Reference

| Property | LocalStorage | SessionStorage |
|---|---|---|
| Persistence | Until explicitly cleared | Tab lifetime (gone on tab close) |
| Scope | Origin-wide (all tabs + windows) | Single tab/window |
| Capacity | ~5–10MB (origin-dependent) | ~5–10MB (origin-dependent) |
| API | Synchronous, string-only | Synchronous, string-only |
| Cross-tab events | `storage` event fires in other tabs | No — scoped to one tab |
| Accessible from Workers | No | No |

---

## What Is This?

`localStorage` and `sessionStorage` are synchronous, string-based key-value stores scoped to an origin. They're part of the Web Storage API (HTML5, 2009) and share the same interface:

```js
// Write
localStorage.setItem('theme', 'dark');
sessionStorage.setItem('form-draft', JSON.stringify({ name: 'Alice' }));

// Read
const theme = localStorage.getItem('theme');         // 'dark'
const draft = JSON.parse(sessionStorage.getItem('form-draft')); // { name: 'Alice' }

// Delete
localStorage.removeItem('theme');
sessionStorage.clear(); // wipe all keys

// Iterate
for (let i = 0; i < localStorage.length; i++) {
  const key = localStorage.key(i);
  console.log(key, localStorage.getItem(key));
}
```

The crucial difference is **persistence and scope**:
- `localStorage` persists until cleared, and is shared across all tabs and windows of the same origin
- `sessionStorage` is scoped to the specific tab (browsing context), and is wiped when the tab closes

> **Check yourself:** If you open two tabs of the same site and write to `sessionStorage` in Tab A, can Tab B read it?

---

## Why They Exist

Before Web Storage, the only client-side persistence options were:
- **Cookies**: sent on every HTTP request (overhead), size-limited to 4KB, originally designed for server communication
- **Document.domain tricks**: fragile cross-origin hacks
- **Flash LSOs**: plugin-dependent, dying

Web Storage gave JS a simple, large-capacity key-value store intended purely for client-side use. No cookies are set, no HTTP headers added. It solved the use case of "remember my preference / draft without polluting my requests."

---

## How It Works

### Synchronous and blocking

Both APIs are entirely synchronous — `getItem` and `setItem` block the main thread. In practice this is rarely a problem (reads/writes are fast, storage is in RAM), but it means:
- You cannot use them in Web Workers (they'd block the worker's thread, and Workers don't have access anyway)
- Long iteration over many keys can cause slight jank on low-end devices
- There's no async alternative — if you need async, use IndexedDB

### String-only storage

Values must be strings. Non-strings are coerced:

```js
localStorage.setItem('count', 42);     // stored as '42'
localStorage.setItem('flag', true);    // stored as 'true'
localStorage.setItem('obj', { a: 1 }); // stored as '[object Object]' — common bug

// Correct pattern for objects:
localStorage.setItem('user', JSON.stringify({ name: 'Alice', age: 30 }));
const user = JSON.parse(localStorage.getItem('user'));
```

Forgetting `JSON.stringify`/`JSON.parse` is the most common bug.

### Capacity

Browsers typically allocate 5MB per origin for localStorage (some grant up to 10MB). Exceeding the quota throws a `DOMException` with name `QuotaExceededError`:

```js
try {
  localStorage.setItem('big', 'x'.repeat(10_000_000));
} catch (e) {
  if (e.name === 'QuotaExceededError') {
    console.warn('Storage full');
  }
}
```

Always wrap large writes in a try/catch.

### `sessionStorage` scoping in detail

"Session" means browsing session for the specific tab:
- Opening a new tab (even to the same URL) creates a **separate** `sessionStorage`
- Duplicating a tab copies `sessionStorage` at the moment of duplication, but from then on the two tabs have independent copies
- `sessionStorage` survives page refreshes and navigations within the tab
- `sessionStorage` is destroyed when the tab closes, or when the window is closed (for non-tabbed contexts)

This makes `sessionStorage` ideal for form drafts, multi-step wizard state, and unsaved changes — data that should not leak to other tabs and should be discarded when the user is done.

---

## The `storage` Event

When `localStorage` changes in one tab, other tabs on the same origin receive a `storage` event:

```js
// In Tab B — fired when Tab A modifies localStorage
window.addEventListener('storage', (event) => {
  console.log(event.key);       // changed key
  console.log(event.oldValue);  // previous value
  console.log(event.newValue);  // new value
  console.log(event.url);       // URL of the page that made the change
  console.log(event.storageArea); // localStorage or sessionStorage reference
});
```

The tab that makes the change does NOT receive the event — it only fires in other tabs. This is the simplest cross-tab communication mechanism (simpler than Shared Workers or BroadcastChannel for basic use cases like "notify other tabs that the user logged out").

`sessionStorage` does not fire `storage` events across tabs — it's tab-scoped by design.

---

## When to Use (and Not Use)

### Use `localStorage` for:
- User preferences (theme, language, display density)
- Non-sensitive settings that should persist across sessions
- Caching derived data that's expensive to recompute (UI state, feature flags)
- Simple "remember me" state that supplements a cookie session

### Use `sessionStorage` for:
- Form draft data (auto-save mid-session)
- Multi-step wizard progress
- Scroll position within a complex page
- Tab-specific selection state

### Don't use either for:
- **Auth tokens / sensitive data**: both are accessible via `document.cookie`-equivalent JS (`localStorage.getItem('token')`). XSS can read them. Use `HttpOnly` cookies for session tokens.
- **Large binary data**: they're string-only and have 5–10MB limits. Use IndexedDB or Cache API for images, files, or large structured data.
- **Data that Workers need**: Workers can't access either API.
- **Offline-first apps**: localStorage lacks transactions, structured queries, and size for serious offline data. IndexedDB is the right tool.

---

## Comparison with Other Storage

| Storage | Async | Size limit | Worker access | Structured data | Auto-sent with requests |
|---|---|---|---|---|---|
| Cookie | Sync | 4KB | No | No (string) | Yes |
| localStorage | Sync | ~5–10MB | No | No (string) | No |
| sessionStorage | Sync | ~5–10MB | No | No (string) | No |
| IndexedDB | Async | Varies (GBs) | Yes | Yes (most types) | No |
| Cache API | Async | Varies (GBs) | Yes | Request/Response | No |

---

## Gotchas

**1. localStorage is accessible by any JS on the page**
Including third-party scripts. If you load analytics, ads, or widgets, they can read (and write) your localStorage. Never store session tokens, PII, or sensitive state there.

**2. Incognito/private mode has separate, session-scoped storage**
`localStorage` in an incognito tab persists only for that incognito session. Some browsers (Safari) have historically thrown on writes in private mode when the quota is 0. Always wrap writes in try/catch.

**3. Synchronous writes block layout in edge cases**
Writing a large string (near the quota limit) on every keystroke — e.g., autosaving a rich-text editor — can block the main thread. Debounce autosaves or move large serialization off the critical path.

**4. `storage` event does not fire in the originating tab**
A common bug: developers attach a `storage` listener expecting to react to their own writes. It doesn't fire locally — only in other tabs. Use a custom event or callback for same-tab reactions.

**5. `JSON.parse(null)` returns `null`**
`localStorage.getItem('missing-key')` returns `null` (not `undefined`). `JSON.parse(null)` returns `null` without throwing. So `JSON.parse(localStorage.getItem('user'))` when 'user' is not set gives `null`, not an error — which can cause subtle `null.property` bugs downstream.

**6. Storage is synchronous with the rendering pipeline**
Heavy `localStorage` use on every frame (animation tick reading preferences) is a performance antipattern. Cache localStorage values in JS variables; read from storage only on load or on explicit user actions.

---

## Interview Questions

**Q (High): What is the difference between localStorage and sessionStorage?**

Answer: Both are synchronous, string-based key-value stores scoped to an origin. The difference is scope and persistence. `localStorage` persists until explicitly cleared and is shared across all tabs and windows of the same origin — all tabs see the same data. `sessionStorage` is scoped to a single browsing context (tab or window) and is destroyed when that context closes. Two tabs of the same site have completely independent `sessionStorage`. Use `localStorage` for persistent user preferences; use `sessionStorage` for transient per-tab state (form drafts, wizard progress) that shouldn't leak between tabs.

**Q (High): Why shouldn't you store auth tokens in localStorage?**

Answer: localStorage is accessible to any JavaScript running on the page — including third-party scripts, injected ads, analytics snippets, or XSS payloads. An XSS attacker can read `localStorage.getItem('token')` and exfiltrate it. Auth tokens stored in localStorage are effectively exposed to any script execution on your origin. The secure alternative is `HttpOnly` cookies — the browser sends them automatically with requests but JavaScript cannot read them, preventing exfiltration even if an attacker achieves script injection. The cost is that cookies require CSRF protection (`SameSite`), but that's a narrower attack surface than all-JS-can-read localStorage.

**Q (Medium): How does the `storage` event work and what is it useful for?**

Answer: The `storage` event fires on `window` in all other tabs/windows of the same origin when `localStorage` changes. It does not fire in the tab that made the change. The event contains `key`, `oldValue`, `newValue`, `url` (of the mutating page), and `storageArea`. It's useful for simple cross-tab synchronization: detecting logout in another tab, syncing theme changes, or propagating feature flag updates. For more complex cross-tab state, BroadcastChannel or Shared Workers are better choices. `sessionStorage` changes do not fire `storage` events because sessionStorage is per-tab by design.

**Q (Medium): What are the capacity limits of Web Storage and what happens when you exceed them?**

Answer: Browsers typically allocate 5MB per origin for localStorage (some up to 10MB). Exceeding the limit throws a `DOMException` with `name === 'QuotaExceededError'`. Unlike most storage APIs, there's no quota query API — you can only discover the limit by hitting it. Best practice: always wrap large `setItem` calls in try/catch, and consider whether data should be in IndexedDB if you're approaching multi-megabyte storage.

**Q (Low): If a user duplicates a tab, what happens to sessionStorage?**

Answer: The duplicated tab receives a copy of the original tab's `sessionStorage` at the moment of duplication — so the new tab starts with the same keys and values. From that point on, the two tabs have completely independent copies: changes in one don't affect the other, and there's no `storage` event between them. This is sometimes surprising — developers expect sessionStorage to be "per-session" in a strict sense, but duplication transfers the state. For truly isolated per-tab state, you'd need to generate a unique tab ID and prefix all storage keys with it.

---

## Self-Assessment

- [ ] Explain the difference between localStorage and sessionStorage scope without looking at notes
- [ ] Describe what happens to sessionStorage when a tab is duplicated vs. closed
- [ ] Explain why auth tokens in localStorage are a security risk compared to HttpOnly cookies
- [ ] Write a correct pattern for storing and retrieving an object from localStorage
- [ ] Describe the `storage` event: when it fires, where it fires, and what it contains

---
*Next: IndexedDB — the browser's true database, with async transactions, object stores, indexes, and cursor iteration for structured, large-scale offline data.*
