# Heap Profiling & DevTools Memory Snapshots

## Quick Reference

| Tool | What It Shows | When to Use |
|---|---|---|
| Heap Snapshot | All live objects at a point in time | Identify what's alive and how much it costs |
| Allocation Timeline | Objects allocated per time window | Find which code paths allocate most |
| Allocation Sampling | Statistical allocation profile | Low-overhead production-safe profiling |
| `performance.memory` | JS heap size (Chromium-only) | Basic leak trending in RUM |

---

## Why Heap Profiling Is a Skill

Removing console.log calls and guessing at leak sources is not a debugging strategy. Browser DevTools expose the full heap object graph — you can see exactly which objects exist, how much memory they retain, and what's keeping them alive. The skill is knowing how to read that data.

Two key concepts before diving in:

- **Shallow size:** memory occupied by the object itself — its own fields and slots. A 10-element array with large objects has a small shallow size.
- **Retained size:** shallow size of the object **plus** the total size of all objects that would be freed if this object were deleted — i.e., everything it keeps alive through reference chains. Retained size is what you care about for leaks.

---

## Taking a Heap Snapshot

**In Chrome DevTools:**
1. Open Memory tab → select "Heap snapshot" → click "Take snapshot"
2. Perform the action you suspect causes a leak
3. Click the GC button (trash icon) to force a collection
4. Take a second snapshot

The raw snapshot view shows every constructor in your app. Sort by "Retained Size" descending to see what's consuming the most memory.

**Filter for "Detached":**

Type "Detached" in the filter box. Any results are DOM nodes that have been removed from the document but are kept alive by JavaScript references.

```
Retainers panel example:

HTMLDivElement (detached)
  ← cache["user-123"]         (Map entry)
    ← UserService              (class instance)
      ← window.services        (global)
```

The "Retainers" panel shows the reference chain from the object back to a GC root. This tells you exactly what's keeping the node alive.

---

## Comparing Snapshots

The most powerful leak-detection technique: **snapshot comparison**.

1. Snapshot 1 — baseline
2. Do the action (navigate to a route, open a modal, add items)
3. Undo the action (navigate back, close the modal)
4. Force GC → Snapshot 2
5. In snapshot 2's view, change "Summary" to **"Comparison"** against snapshot 1

The comparison shows:
- `# New`: objects created between snapshots that weren't collected
- `# Deleted`: objects that were created before snapshot 1 and are now gone
- `Delta`: net change in object count

Any constructor with a positive `# New` after you've "undone" the action is a leak candidate.

```
Constructor              # New    # Deleted    Delta    Alloc. Size
HTMLDivElement            24           0        +24       96 KB
Modal                      1           0         +1        4 KB
EventListener             12           0        +12        2 KB
```

This snapshot comparison tells you: 24 div elements, 1 Modal instance, and 12 event listeners were allocated and not collected after closing the modal.

---

## Allocation Timeline

The Allocation Timeline records allocations over time. It's useful when you want to find **which code path** is causing allocations, not just what's allocated.

1. Memory tab → "Allocation instrumentation on timeline" → Start
2. Perform operations
3. Stop
4. Blue bars in the timeline = live allocations (not yet collected)
5. Grey bars = collected

Click a blue bar to see what was allocated in that time window and the call stack that triggered it.

**Use case:** finding unexpected allocations in hot render paths:

```typescript
// This creates a new array on every render — visible in the allocation timeline
function Component({ items }: { items: string[] }) {
  const processed = items.map(x => x.toUpperCase()); // new array each time
  return <List items={processed} />;
}

// Fix: useMemo to prevent unnecessary allocation
import { useMemo } from 'react';

function ComponentFixed({ items }: { items: string[] }) {
  const processed = useMemo(() => items.map(x => x.toUpperCase()), [items]);
  return <List items={processed} />;
}
```

---

## Allocation Sampling

Allocation sampling provides a statistical profile of where memory is being allocated. Unlike the timeline, it has very low overhead — you can run it longer without affecting the page significantly.

**Use for:** identifying memory-heavy hot paths, not precise object counting.

The output is a flame chart showing allocation by function. Functions at the top of large flame columns are allocating the most memory.

---

## Reading a Heap Snapshot — Key Patterns

**Pattern 1: Growing closure count**

If `(closure)` constructor count grows between snapshots, you have callbacks being allocated and not collected. Expand one, look at the retained objects and the retainer chain.

**Pattern 2: Growing Map or Array count**

If `Map` or `Array` counts grow, you have collections being added to without being cleaned up. Check if any global/module-level collection is being appended to without eviction.

**Pattern 3: "Window" as retainer**

If the retainer chain ends at `Window → some property`, the object is reachable from a global. That means it's intentionally global, or something that should be scoped isn't.

**Pattern 4: `(string)` dominates retained size**

Large string retained sizes often indicate cached template strings, large JSON objects, or logging data accumulating without being released.

---

## `performance.memory` (Chromium only)

```typescript
interface PerformanceMemory {
  usedJSHeapSize: number;    // currently used heap
  totalJSHeapSize: number;   // committed heap (may be larger than used)
  jsHeapSizeLimit: number;   // maximum allowed heap for this context
}

function trackMemory(): void {
  if (!('memory' in performance)) return; // only Chromium

  const { usedJSHeapSize, jsHeapSizeLimit } = (performance as unknown as {
    memory: PerformanceMemory;
  }).memory;

  const usagePercent = (usedJSHeapSize / jsHeapSizeLimit) * 100;
  sendMetric('heap_usage_pct', usagePercent);
}
```

Use this for RUM trending: if `usedJSHeapSize` grows continuously between route navigations without dropping, that's a leak signal you can alert on.

---

## `performance.measureUserAgentSpecificMemory()`

The newer, cross-browser API for memory measurement (requires COOP/COEP headers):

```typescript
async function measureMemory(): Promise<void> {
  if (!('measureUserAgentSpecificMemory' in performance)) return;

  try {
    const result = await (performance as unknown as {
      measureUserAgentSpecificMemory: () => Promise<{ bytes: number }>;
    }).measureUserAgentSpecificMemory();

    console.log(`Memory used: ${(result.bytes / 1024 / 1024).toFixed(2)} MB`);
  } catch {
    // SecurityError if headers are not set correctly
  }
}
```

---

## A Systematic Leak Investigation Workflow

```
1. Establish baseline
   → Heap snapshot after page load + idle

2. Reproduce the suspected action N times
   → Navigate back and forth, open/close modal 5×, etc.

3. Force GC → take snapshot
   → Compare to baseline

4. Sort comparison by "# New" descending
   → Identify the growing constructor types

5. Expand the top candidates
   → Look at retainer chain → trace to root

6. Identify the retaining reference
   → Map entry? Global? Closure? Event listener?

7. Fix and re-run
   → If the # New delta disappears, the leak is fixed
```

> **Check yourself:** You take two snapshots of the same page state and see that `Map` count is `+50`. What additional information do you need before you can fix the leak?

---

## Self-Assessment

- [ ] I understand the difference between shallow size and retained size
- [ ] I can take two heap snapshots and use the comparison view to find growing object types
- [ ] I know how to use the "Detached" filter to find orphaned DOM nodes
- [ ] I can read the Retainers panel to trace why an object is still alive
- [ ] I understand when to use Allocation Timeline vs. Allocation Sampling
- [ ] I know the limitations of `performance.memory` (Chromium-only, not precise)

---

## Interview Q&A

**Q: What is "retained size" and why is it more useful than "shallow size" for debugging leaks?** `High`

Shallow size is the memory occupied by the object's own data — its fields and slots. Retained size is shallow size plus all the memory that would be freed if that object were collected — everything it keeps alive through references. A small object with a large retained size is a common leak pattern: a tiny event listener holding a reference to a large component tree. Retained size is what you sort by when debugging because it shows the real memory cost of keeping an object alive.

---

**Q: Walk me through how you would find a memory leak in a modal component.** `High`

Take a baseline heap snapshot. Open the modal, interact with it, then close it. Force a GC collection. Take a second snapshot and switch to Comparison view. Look for any constructors with a positive `# New` delta — particularly the modal's component class, its DOM nodes, and any event listeners. Expand the leaking objects and look at the Retainers panel to trace the reference chain from the object back to a GC root. That chain identifies exactly which reference is keeping the object alive.

---

**Q: What does it mean when the Retainers panel shows "Window → globalCache → element"?** `Medium`

The element is reachable from the global `window` object through a property called `globalCache`. That's why it can't be collected — the GC traces from `window`, finds `globalCache`, and finds the element. To fix the leak, the element must be removed from `globalCache` when it's no longer needed. The retainer chain is the "why it's alive" — fixing the leak means breaking that chain.

---

**Q: What's the difference between Allocation Timeline and Allocation Sampling?** `Medium`

Allocation Timeline records every allocation in a time window, marks which ones were collected (grey bars) vs. still live (blue bars), and lets you inspect the exact call stack for any allocation. It's precise but adds significant overhead — not suitable for long sessions. Allocation Sampling uses statistical sampling to build a flame chart of allocation by function with much lower overhead. Use Timeline when you want to inspect specific allocations and their call stacks; use Sampling when you want an overview of which functions are allocating the most over a longer period.

---

**Q: Is `performance.memory.usedJSHeapSize` reliable for detecting leaks?** `Low`

It's useful for trending, not for precise measurement. It's only available in Chromium-based browsers. The values reflect the heap state at the moment of the call, which may or may not reflect a recent GC. Because GC is non-deterministic, you need to sample repeatedly over time and look for a consistently upward trend rather than a snapshot value. For production monitoring, send it as a RUM metric and alert on sustained growth across route navigations. For reliable leak detection in DevTools, use heap snapshots and the comparison view.
