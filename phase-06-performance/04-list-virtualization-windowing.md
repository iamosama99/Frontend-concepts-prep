# List Virtualization & Windowing

## Quick Reference

| Approach | DOM nodes created | Scroll performance | Memory | Use case |
|---|---|---|---|---|
| Full render | All N items | Degrades with N | O(N) DOM nodes | Small lists (< 100 items) |
| Pagination | Page size items | Constant | O(page) DOM nodes | Structured browsing, SEO-important content |
| Infinite scroll | Grows with scrolling | Degrades without recycling | O(scroll depth) | Social feeds (without virtualization) |
| Virtualization (windowing) | ~viewport items only | Constant regardless of N | O(viewport) DOM nodes | Large data tables, long feeds, any list > 500 items |

---

## What Is This?

List virtualization (also called windowing) is a rendering technique where only the items currently visible in the viewport — plus a small overscan buffer — exist in the DOM. Items above and below the viewport are not rendered; as the user scrolls, items entering the viewport are mounted and items leaving it are unmounted (or recycled). The total DOM node count stays roughly constant regardless of how many items the list contains.

The key insight: the browser does not need to lay out what it cannot see. Rendering 10,000 rows forces the browser to create 10,000 DOM nodes, compute their layout, and store them in memory — even though the user sees only 20 at a time. Virtualization cuts this to ~30–40 nodes regardless of list length.

> **Check yourself:** A virtualized list of 10,000 items maintains a constant number of DOM nodes. How does it know what content to put in each node as the user scrolls?

---

## Why Does It Exist?

DOM nodes are expensive in three ways:

1. **Memory**: Each DOM element is a C++ object in the browser's memory. Thousands of nodes means megabytes of memory for elements the user will never see.

2. **Layout**: The browser lays out elements in document order. A list of 10,000 rows forces the layout engine to process 10,000 elements and their children on initial paint, even though only ~20 fit in the viewport.

3. **Style recalculation**: Every time a class changes or a CSS variable updates, the browser must check which elements are affected. More nodes = more work per style change.

On a mid-range mobile device, rendering a list of 1,000 complex items can take 2–4 seconds of blocking layout work. With virtualization, that work is constant — proportional to viewport size, not list length.

---

## How It Works

### Core Mechanism

A virtualized list needs three pieces of information:

1. The total list height (to make the scrollbar accurate)
2. The scroll position (to know which items are in view)
3. Each item's height (to calculate which indices are visible at a given scroll offset)

```
Total container height = sum of all item heights (or count × fixed height)
Visible window = items whose position overlaps [scrollTop, scrollTop + containerHeight]
Rendered items = visible window + overscan buffer (e.g., 3 items above and below)
```

The container has a fixed height with `overflow: auto`. Inside it, a "spacer" element has its height set to the total list height so the scrollbar is accurate. The visible items are positioned absolutely within this spacer using `transform: translateY(offset)` for each item.

```
┌─────────────────────────────────┐ ← container (fixed height, overflow: auto)
│ [spacer: totalHeight px]        │
│                                 │
│   item 23 (translateY: 2300px)  │ ← only rendered items
│   item 24 (translateY: 2400px)  │
│   item 25 (translateY: 2500px)  │
│   item 26 (translateY: 2600px)  │
│                                 │
└─────────────────────────────────┘
```

As the user scrolls, a scroll event fires, the component recalculates which indices are visible, and React re-renders with the new set of items at their correct positions.

---

### Fixed-Height Items with `react-window`

`react-window` by Brian Vaughn is the standard lightweight virtualization library. The simplest case is a list where every item has the same height.

```js
import { FixedSizeList } from 'react-window';

const items = Array.from({ length: 10000 }, (_, i) => ({ id: i, name: `Item ${i}` }));

function Row({ index, style }) {
  const item = items[index];
  // style MUST be applied — it contains the position/size for this item
  return (
    <div style={style} className="list-row">
      {item.name}
    </div>
  );
}

function VirtualList() {
  return (
    <FixedSizeList
      height={600}          // visible container height in px
      width="100%"
      itemCount={items.length}
      itemSize={50}         // each row is exactly 50px tall
    >
      {Row}
    </FixedSizeList>
  );
}
```

**The `style` prop is mandatory.** It contains `position: absolute`, `top`, `height`, `width` that position the item correctly within the virtualized container. Omitting it breaks the layout.

---

### Variable-Height Items with `react-window`

When items have different heights, you need `VariableSizeList`. You provide a `getItemSize` function that returns the height for a given index.

```js
import { VariableSizeList } from 'react-window';

const itemHeights = items.map((item) =>
  item.isExpanded ? 120 : 50
);

function getItemSize(index) {
  return itemHeights[index];
}

function Row({ index, style }) {
  return (
    <div style={style}>
      <ItemContent item={items[index]} />
    </div>
  );
}

const listRef = React.createRef();

function VirtualList() {
  return (
    <VariableSizeList
      ref={listRef}
      height={600}
      width="100%"
      itemCount={items.length}
      itemSize={getItemSize}
      estimatedItemSize={50}  // helps initial scroll estimate
    >
      {Row}
    </VariableSizeList>
  );
}
```

**When item heights change** (e.g., an accordion expands), you must call `listRef.current.resetAfterIndex(changedIndex)` — otherwise the cached position calculations are wrong and items appear at the wrong scroll offset.

```js
function handleExpand(index) {
  itemHeights[index] = 120;
  listRef.current.resetAfterIndex(index);
}
```

---

### `react-virtual` (TanStack Virtual)

TanStack Virtual is a headless, framework-agnostic virtualization hook. It gives you raw numbers (start index, end index, item positions) and you build the DOM structure yourself. This is more flexible than `react-window` for complex layouts like grids or masonry.

```js
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';

function VirtualList({ items }) {
  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,  // initial estimate; measures actual after render
    overscan: 5,             // extra items above and below viewport
  });

  return (
    <div
      ref={parentRef}
      style={{ height: '600px', overflowY: 'auto' }}
    >
      {/* Spacer div — sets total scroll height */}
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            data-index={virtualItem.index}
            ref={virtualizer.measureElement}  // measures actual height after render
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            <Row item={items[virtualItem.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

TanStack Virtual auto-measures item heights after they render (via `measureElement`), so you don't need to know heights upfront — it starts with estimates and corrects as items become visible.

---

### Virtualized Grids

For 2D grids (spreadsheets, image galleries), both `react-window`'s `FixedSizeGrid`/`VariableSizeGrid` and TanStack Virtual's grid mode support virtualizing along both axes.

```js
import { FixedSizeGrid } from 'react-window';

function Cell({ columnIndex, rowIndex, style }) {
  return (
    <div style={style}>
      Row {rowIndex}, Col {columnIndex}
    </div>
  );
}

function DataGrid() {
  return (
    <FixedSizeGrid
      columnCount={50}
      columnWidth={100}
      height={500}
      rowCount={1000}
      rowHeight={35}
      width={800}
    >
      {Cell}
    </FixedSizeGrid>
  );
}
```

---

## The Overscan Buffer

Rendering only the visible items creates a noticeable flash of blank space during fast scrolling — the user scrolls faster than React can mount new items. The `overscan` option (or `overscanCount` in `react-window`) renders extra items above and below the visible area as a buffer.

- Too small (0–1): blank areas appear during fast scrolling
- Too large (> 10): negates the performance benefit by rendering too many items

A value of 3–5 is typical. Increase it if users report blank areas; decrease it if profiling shows too many off-screen renders.

---

## Measuring Without a Library

If you need virtualization without a dependency, the core calculation for fixed-height items:

```js
function getVisibleRange(scrollTop, containerHeight, itemHeight, itemCount) {
  const startIndex = Math.floor(scrollTop / itemHeight);
  const visibleCount = Math.ceil(containerHeight / itemHeight);
  const endIndex = Math.min(startIndex + visibleCount + 1, itemCount - 1);

  return { startIndex, endIndex };
}

// Usage in a scroll handler (throttled)
container.addEventListener('scroll', throttle(() => {
  const { startIndex, endIndex } = getVisibleRange(
    container.scrollTop,
    container.clientHeight,
    50,      // itemHeight
    items.length
  );

  renderVisibleItems(startIndex, endIndex);
}, 16)); // ~60fps
```

---

## When Not to Virtualize

Virtualization adds complexity and has trade-offs:

- **Accessibility**: Screen readers rely on the DOM to enumerate list items. A virtualized list that removes items from the DOM can confuse screen reader navigation. Use `aria-rowcount` and `aria-rowindex` to communicate the full list size.
- **Keyboard navigation**: If items not in the viewport are not in the DOM, keyboard focus cannot reach them without custom handling.
- **SEO**: Content not in the DOM is not indexed. For SEO-important lists (product catalogs), server-render the visible page and virtualize only the client-side experience.
- **Small lists**: The overhead of virtualization (scroll listeners, position calculations, React re-renders on scroll) exceeds the benefit for lists under ~100 items on desktop.

---

## Gotchas

**Forgetting to apply the `style` prop.** The `style` prop from `react-window` contains the absolute positioning for each item. If you spread it onto a wrapper div but your actual content div doesn't receive the dimensions, items will overlap or the container will collapse.

**Item height changes without `resetAfterIndex`.** With `VariableSizeList`, the library caches computed offsets. If an item's height changes (accordion expand, image load) without calling `resetAfterIndex`, all subsequent items will be positioned at incorrect offsets.

**Scroll position reset on data update.** If the items array changes identity (e.g., a new array reference from a filter), `react-window` may reset the scroll position to 0. Use stable array references or `itemKey` props to help the library maintain scroll position.

**Measuring variable-height items before render.** `VariableSizeList` requires knowing heights before rendering. If your items contain images or dynamic content that affects height, you need either a fixed estimated height or TanStack Virtual's `measureElement` approach that measures after render.

---

## Interview Questions

**Q (High): Explain how list virtualization works mechanically. What is always in the DOM, and what changes as the user scrolls?**
Answer: A virtualized list has three DOM elements: a scroll container with a fixed height and `overflow: auto`; a spacer element whose height equals the total height of all items (so the scrollbar is accurate); and the rendered item elements, which are absolutely positioned within the spacer. At any given scroll position, only the items whose vertical range overlaps the visible viewport (plus an overscan buffer of a few items above and below) are in the DOM. As the user scrolls, a scroll event fires, the component calculates the new visible index range based on `scrollTop / itemHeight`, and React mounts newly-visible items and unmounts items that have left the viewport. The DOM node count stays constant — roughly `visibleItems + overscan * 2` — regardless of total list length.
The trap: Saying items are "hidden" or set to `display: none`. They are not in the DOM at all — that's the point.

**Q (High): You're building a comments thread where each comment can expand to show replies, making its height variable. Which library approach would you use and what do you need to handle when a comment expands?**
Answer: Use `VariableSizeList` from `react-window` or TanStack Virtual's variable mode. With `VariableSizeList`, you provide a `getItemSize(index)` function. When a comment expands, you update the height in your height map and call `listRef.current.resetAfterIndex(expandedIndex)` — this clears the cached offset calculations for all items from that index onward and triggers a recalculation. Without `resetAfterIndex`, comments after the expanded one appear at wrong vertical positions. TanStack Virtual's `measureElement` ref approach is simpler: it auto-measures actual rendered heights and corrects positions after each render, so you don't need to track heights manually or call reset methods — it handles the correction cycle automatically.
The trap: Not knowing about `resetAfterIndex` or thinking you can just re-render the parent component.

**Q (Medium): What are the accessibility problems with list virtualization, and how do you mitigate them?**
Answer: Screen readers enumerate lists by reading DOM elements in order. A virtualized list where 9,980 out of 10,000 items are not in the DOM means a screen reader will announce only ~20 items, not 10,000. Users navigating by keyboard or screen reader cannot access items not currently rendered. Mitigations: use `aria-rowcount` and `aria-rowindex` on table rows so screen readers announce the correct total count even when rows aren't all rendered; use `role="list"` and `role="listitem"` with `aria-setsize` and `aria-posinset`; ensure keyboard focus management explicitly handles scrolling to and rendering the focused item. For truly critical accessible lists (navigation menus, form field lists), consider pagination or server rendering instead. Virtualization and full accessibility are in tension — document the trade-off explicitly.
The trap: Not knowing the ARIA attributes (`aria-rowcount`, `aria-setsize`, `aria-posinset`) that communicate list size to assistive technology.

**Q (Medium): When would you choose TanStack Virtual over `react-window`?**
Answer: `react-window` is smaller and simpler — best for straightforward fixed-height lists and grids where performance is critical and you don't need framework flexibility. TanStack Virtual is headless (gives you numbers, you own the DOM) and handles dynamic/unknown heights automatically via `measureElement` — it measures actual rendered heights and corrects positions. Choose TanStack Virtual when: items have truly dynamic heights (images, expandable content) and you don't want to maintain a height map; you need the flexibility to build custom layouts (masonry, horizontal scrolling); or you are working in a non-React environment (it is framework-agnostic). TanStack Virtual's auto-measurement loop adds a paint cycle after initial render as it measures and repositions, which can cause a brief visual jump — `react-window` avoids this by requiring heights upfront.
The trap: Thinking TanStack Virtual always has better performance. The auto-measurement loop is a trade-off.

**Q (Low): A virtualized list of 50,000 items scrolls smoothly on desktop but users report blank rows during fast scrolling on mobile. What do you adjust?**
Answer: Increase the `overscan` value. The overscan buffer pre-renders items above and below the visible viewport so fast scrolling has a margin before hitting unrendered content. The default (3–5) may be insufficient on mobile where scroll velocity is higher and React render time is slower due to weaker CPUs. Increase to 10–15 and re-test. If blank rows persist, the issue may be that the scroll event handling is throttled too aggressively — ensure it runs on every animation frame (`requestAnimationFrame` or the browser's native scroll handling). Also check that the items themselves are not doing expensive computation during render — simplify or memoize item components.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Describe the three DOM elements that exist in a virtualized list and explain why the total scroll height is correct
- [ ] Implement a `FixedSizeList` with `react-window` including the required `style` prop
- [ ] Explain what `resetAfterIndex` does and when you must call it
- [ ] Describe TanStack Virtual's `measureElement` approach and contrast it with `react-window`'s height requirement
- [ ] List two accessibility problems with virtualization and the ARIA attributes that mitigate them
- [ ] State when virtualization is not worth the complexity

---
*Next: Tree Shaking & Dead Code Elimination — code splitting divides bundles at route boundaries; tree shaking eliminates unused exports within each bundle.*
