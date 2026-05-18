# Memory Leaks: Detached DOM Nodes, Closures, Event Listeners

## Quick Reference

| Leak Type | Root Cause | Detection Signal | Fix |
|---|---|---|---|
| Detached DOM nodes | JS reference keeps removed nodes alive | "Detached HTMLElement" in heap snapshot | Null the reference on removal |
| Closure capture | Long-lived callback captures large scope | Growing retained size on closures | Capture only what's needed |
| Event listeners | Listener holds reference to removed element | Listener count growing, node not GC'd | `removeEventListener` on cleanup |
| Timer/interval | setInterval callback keeps closure alive | Retained objects grow across time | `clearInterval` on unmount |

---

## Why Memory Management Matters in Frontend

JavaScript has a garbage collector (GC), but GC can only collect objects it can prove are unreachable. The browser's definition of "reachable" is broader than you might expect: anything referenced from a **GC root** (global object, current call stack, active DOM) stays alive.

The consequence: code that looks like it cleans up after itself often doesn't, because a dangling reference somewhere keeps the entire object graph alive. In long-lived SPAs — where the user never does a full page reload — these leaks compound over time into sluggish tabs, excessive memory warnings, and eventually crashes.

Understanding leaks means understanding GC roots and reference chains, not just calling `remove()` on a DOM node.

---

## How the Garbage Collector Works

The browser uses a **mark-and-sweep** algorithm:

1. Start from all GC roots (window, document, active stack frames, Web Worker global).
2. Recursively mark every object reachable from those roots.
3. Sweep: collect everything that wasn't marked.

An object leaks when it remains reachable from a root — even if your application logic considers it "done."

```
GC Root (window)
  └── myCache (object property)
        └── listener (function)
              └── element (DOM node you thought you removed)
```

Even though `element` was removed from the DOM, it's still reachable through `window.myCache.listener`. The GC won't collect it.

---

## Leak Pattern 1: Detached DOM Nodes

**What it is:** A DOM node is removed from the live document tree but a JavaScript reference keeps it (and its entire subtree) alive.

**Why it leaks:** The DOM tree is just a data structure. Removing a node from the tree severs the document's reference to it, but any JS variable, closure, or data structure that still points to that node holds it in memory.

```typescript
class UserList {
  private cache = new Map<string, HTMLElement>();

  render(users: string[]): void {
    const container = document.getElementById('list')!;

    users.forEach(user => {
      const el = document.createElement('li');
      el.textContent = user;
      container.appendChild(el);
      this.cache.set(user, el); // ← stores reference
    });
  }

  clear(): void {
    const container = document.getElementById('list')!;
    container.innerHTML = ''; // removes from DOM
    // Bug: this.cache still holds all the removed <li> elements
    // They appear as "Detached HTMLElement" in heap snapshots
  }

  clearFixed(): void {
    const container = document.getElementById('list')!;
    container.innerHTML = '';
    this.cache.clear(); // ← also clear the JS reference
  }
}
```

**Detection:** In Chrome DevTools Memory tab, take a heap snapshot and filter for "Detached". Any detached node that appears is a leak.

> **Check yourself:** After calling `container.innerHTML = ''`, are child elements eligible for GC? What determines whether they are?

---

## Leak Pattern 2: Closures Capturing Large Scope

**What it is:** A long-lived callback (event handler, timer, promise chain) was created inside a function that had large variables in scope. Those variables are captured by the closure and can't be released.

**Why it leaks:** JavaScript closures capture the entire lexical environment, not just the variables they actually use. If a callback lives longer than the data it was created with, all that data stays alive.

```typescript
function setupAnalytics(): void {
  const reportingData = new Array(100_000).fill({ event: 'click', meta: '...' });
  // reportingData is 100k items we need only at setup time

  document.addEventListener('visibilitychange', () => {
    // This handler runs for the life of the page.
    // Even if it never references reportingData, the closure still
    // holds a reference to it because it was in scope when the
    // function was created.
    sendPing();
  });
}

// Fix: extract the callback so it doesn't close over the large array
function setupAnalyticsFixed(): void {
  const reportingData = new Array(100_000).fill({ event: 'click', meta: '...' });
  processReportingData(reportingData); // consume it, let it go

  document.addEventListener('visibilitychange', handleVisibility);
}

function handleVisibility(): void {
  sendPing(); // no closure over large data
}
```

**The subtle case — only part of the scope is used:**

```typescript
function makeHandler(bigData: object[], id: string): () => void {
  // The closure captures `bigData` even though only `id` is used.
  return () => console.log(`Handler for ${id}`);
}

// Fix: capture only what's needed
function makeHandlerFixed(id: string): () => void {
  return () => console.log(`Handler for ${id}`);
}
```

> **Check yourself:** A closure that captures a variable but never reads it — does it keep that variable alive? How would you verify this in DevTools?

---

## Leak Pattern 3: Event Listeners on Removed Elements

**What it is:** An event listener is added to a DOM node, the node is later removed from the document, but the listener is never removed. The listener holds a reference to both the callback and the element.

```typescript
class Modal {
  private element: HTMLElement;
  private handleKeydown: (e: KeyboardEvent) => void;

  constructor() {
    this.element = document.createElement('div');
    this.element.className = 'modal';

    // Store the bound handler so we can remove it later
    this.handleKeydown = this.onKeydown.bind(this);
    document.addEventListener('keydown', this.handleKeydown);
  }

  private onKeydown(e: KeyboardEvent): void {
    if (e.key === 'Escape') this.close();
  }

  close(): void {
    this.element.remove();
    // Bug: listener on `document` still holds reference to `this`
    // The entire Modal instance is retained
  }

  closeFixed(): void {
    this.element.remove();
    document.removeEventListener('keydown', this.handleKeydown); // ← required
  }
}
```

**The `AbortController` pattern — cleaner cleanup:**

```typescript
class ModalWithAbort {
  private controller = new AbortController();
  private element: HTMLElement;

  constructor() {
    this.element = document.createElement('div');
    const { signal } = this.controller;

    document.addEventListener('keydown', (e) => {
      if (e.key === 'Escape') this.close();
    }, { signal }); // ← listener auto-removed when signal aborts

    document.addEventListener('click', this.handleOutsideClick, { signal });
  }

  private handleOutsideClick = (e: MouseEvent): void => {
    if (!this.element.contains(e.target as Node)) this.close();
  };

  close(): void {
    this.controller.abort(); // removes all listeners registered with this signal
    this.element.remove();
  }
}
```

`AbortController` is the modern idiom for cleaning up multiple listeners at once. One `abort()` call removes all of them.

---

## Leak Pattern 4: Timers and Intervals

**What it is:** `setInterval` or a recursive `setTimeout` continues running after the component or context that created it is gone.

```typescript
class LiveClock {
  private intervalId: ReturnType<typeof setInterval> | null = null;

  start(): void {
    this.intervalId = setInterval(() => {
      // This callback runs forever unless cleared.
      // It closes over `this`, keeping the entire instance alive.
      this.render();
    }, 1000);
  }

  render(): void {
    document.getElementById('clock')!.textContent = new Date().toISOString();
  }

  // Must be called when the component unmounts
  destroy(): void {
    if (this.intervalId !== null) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }
}

// React equivalent
import { useEffect } from 'react';

function useLiveClock(): void {
  useEffect(() => {
    const id = setInterval(() => {
      // update state...
    }, 1000);

    return () => clearInterval(id); // cleanup on unmount
  }, []);
}
```

---

## React-Specific Leak Patterns

React's useEffect cleanup function is the primary mechanism for preventing leaks:

```typescript
import { useEffect, useState } from 'react';

function useData(url: string) {
  const [data, setData] = useState(null);

  useEffect(() => {
    let cancelled = false; // guard against stale updates

    fetch(url)
      .then(res => res.json())
      .then(json => {
        if (!cancelled) setData(json); // don't update unmounted component
      });

    return () => { cancelled = true; };
  }, [url]);

  return data;
}
```

**Subscription pattern:**

```typescript
import { useEffect } from 'react';

function useStore<T>(store: EventEmitter, event: string, handler: (val: T) => void): void {
  useEffect(() => {
    store.on(event, handler);
    return () => store.off(event, handler); // always unsubscribe
  }, [store, event, handler]);
}
```

---

## Self-Assessment

- [ ] I can explain why removing a node from the DOM doesn't guarantee it's garbage collected
- [ ] I can identify which variables a closure captures and why it captures them
- [ ] I understand why you must store a reference to an event handler to remove it
- [ ] I know the `AbortController` pattern for cleaning up multiple listeners
- [ ] I can use the DevTools heap snapshot to find "Detached HTMLElement" leaks
- [ ] I know why `setInterval` keeps its callback's entire closure alive

---

## Interview Q&A

**Q: What are the three most common sources of memory leaks in browser JavaScript?** `High`

Detached DOM nodes (JS still holds a reference to a removed element), event listeners that outlive their elements (the listener keeps both callback and element alive), and timers/intervals that run past the useful lifetime of the code that created them. A fourth common source is closure capture — a long-lived callback closing over variables that are no longer needed.

---

**Q: Why does removing a node from the DOM not guarantee it gets garbage collected?** `High`

The garbage collector uses mark-and-sweep from GC roots. A DOM node that has been removed from the document tree is no longer reachable via `document`, but if any JavaScript variable, Map entry, closure, or event listener still holds a reference to it, it remains reachable from the `window` root and won't be collected. "Removed from the DOM" is a tree operation, not a memory operation.

---

**Q: How does `AbortController` help with event listener cleanup?** `Medium`

`AbortController` lets you pass a `signal` option to `addEventListener`. When you call `controller.abort()`, all listeners registered with that signal are automatically removed, regardless of how many there are. This avoids the need to store individual handler references for removal and ensures you don't forget any, especially when multiple listeners are added across a component's lifecycle.

---

**Q: Can a closure capture a variable it never references? Does that cause a leak?** `Medium`

Yes, in theory a closure closes over its entire lexical environment. In practice, modern JS engines are smart enough to only retain variables that the function actually references (this is called "closure optimization" or "dead variable elimination"). However, this optimization is not guaranteed, and in some engines or edge cases the entire scope can be retained. The safe approach is to not rely on engine optimization — only capture what you need, preferably by extracting inner callbacks to named functions.

---

**Q: How would you track down a memory leak in a production SPA?** `Medium`

Start with the DevTools Memory tab: take a heap snapshot before the suspected leak, trigger the action, force GC, take another snapshot, then compare them. Look for growing counts of object types between snapshots. Filter for "Detached HTMLElement" to find orphaned nodes. Use the "Allocation timeline" to see where allocations are happening. In production, RUM tools like Sentry can track memory via `performance.measureUserAgentSpecificMemory()` (requires COOP/COEP headers).

---

**Q: What's the risk with `useState` setters called after a component unmounts?** `Low`

Calling a state setter on an unmounted component produces a React warning in development ("Can't perform a React state update on an unmounted component") and in some older React versions could trigger re-renders or errors. The fix is to use a `cancelled` boolean in `useEffect` that is set to `true` in the cleanup function, and guard any asynchronous callback with `if (!cancelled)` before calling the setter. React 18 made some of these errors less severe, but the cleanup pattern is still necessary to prevent logic errors and stale updates.
