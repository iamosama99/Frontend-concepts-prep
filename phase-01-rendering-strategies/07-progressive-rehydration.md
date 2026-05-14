# Progressive Rehydration

## Quick Reference

| Concept | Mechanism | Implication |
|---|---|---|
| Priority-ordered hydration | Above-the-fold and user-interacted components hydrate first | TTI for visible content improves without changing total JS |
| Deferred hydration | Below-fold components hydrate lazily (idle, visible, or on interaction) | Main thread isn't saturated hydrating content the user hasn't seen |
| Viewport-aware hydration | IntersectionObserver gates hydration of off-screen components | Hydration work scales to what the user actually sees |
| Interaction-triggered hydration | Components hydrate on first interaction (hover, focus, click) | Near-zero hydration cost for components the user never interacts with |

## What Is This?

Standard SSR hydrates the entire component tree in one synchronous pass immediately after the JavaScript bundle loads. On a complex page, this can block the main thread for hundreds of milliseconds — the entire page is technically "visible" but completely non-interactive while hydration runs.

Progressive Rehydration breaks this single pass into a prioritized sequence. Components most likely to be interacted with immediately — things visible in the current viewport, already-focused elements, components the user is hovering — hydrate first. Below-fold components, off-screen modals, and infrequently used widgets hydrate later, in the background, when the main thread is less busy.

The end result: Time to Interactive for the critical parts of the page drops significantly, even though the total JavaScript to be executed hasn't changed.

```
Standard hydration:
  JS loads → hydrate entire tree (blocks main thread 400ms) → everything interactive

Progressive rehydration:
  JS loads → hydrate above-fold (blocks ~50ms) → above-fold interactive
           → (idle) hydrate remaining sections progressively
           → eventually everything interactive
```

> **Check yourself:** Progressive rehydration doesn't ship less JavaScript. If total JS is the same, what exactly improves?

## Why Does It Exist?

Hydration is inherently CPU work: React walks the component tree, attaches event handlers, and sets up internal state. On a fast laptop this takes tens of milliseconds. On a mid-range mobile device this can take hundreds. During this period the main thread is blocked — no user interactions, no animations, no scrolling responsiveness.

The insight: hydrating the entire page at once is a false requirement. Users can only interact with what they can see. A footer component hydrated 2 seconds into the page lifecycle affects no user who hasn't scrolled to the footer yet. By deferring that work to idle time, you free up the main thread for the components that actually matter right now.

This is the same logic behind lazy loading images (only load what's in the viewport) applied to JavaScript execution (only hydrate what's in the viewport).

## How It Works

### Viewport-Based Hydration

The most common form: use IntersectionObserver to hydrate components as they enter the viewport.

```javascript
// A generic wrapper that defers hydration until the element is visible
function LazyHydrate({ children, id }) {
  const [hydrated, setHydrated] = useState(false);
  const ref = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setHydrated(true);
          observer.disconnect();
        }
      },
      { rootMargin: '200px' }  // 200px before element enters viewport
    );

    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  if (hydrated) return children;
  
  // Before hydration: render the server HTML as-is (suppress hydration warning)
  return (
    <div ref={ref} suppressHydrationWarning>
      {/* Server-rendered HTML rendered by the browser as static HTML */}
    </div>
  );
}
```

Components wrapped in `LazyHydrate` receive their server-rendered HTML immediately (the static HTML is already in the DOM from SSR) and only start receiving JavaScript interactivity when they scroll into view.

### Idle-Time Hydration

For components that don't need viewport awareness — complex widgets, rich editors, off-screen modals — schedule hydration during browser idle time:

```javascript
function IdleHydrate({ children }) {
  const [hydrated, setHydrated] = useState(false);

  useEffect(() => {
    if ('requestIdleCallback' in window) {
      const id = requestIdleCallback(() => setHydrated(true), { timeout: 2000 });
      return () => cancelIdleCallback(id);
    }
    // Fallback for browsers without rIAF
    const id = setTimeout(() => setHydrated(true), 100);
    return () => clearTimeout(id);
  }, []);

  return hydrated ? children : <StaticShell />;
}
```

### Interaction-Triggered Hydration

The most aggressive form: hydration only begins when a user first interacts with a component. Before that, the component is static HTML. On hover/focus/click, JavaScript loads and hydrates.

```javascript
function OnInteractionHydrate({ children, events = ['hover', 'focus'] }) {
  const [hydrated, setHydrated] = useState(false);
  const ref = useRef(null);

  useEffect(() => {
    const el = ref.current;
    if (!el || hydrated) return;

    const hydrate = () => setHydrated(true);
    const eventMap = {
      hover: ['mouseenter', 'touchstart'],
      focus: ['focusin'],
      click: ['click'],
    };

    const domEvents = events.flatMap(e => eventMap[e] || []);
    domEvents.forEach(event => el.addEventListener(event, hydrate, { once: true }));
    return () => domEvents.forEach(event => el.removeEventListener(event, hydrate));
  }, [hydrated, events]);

  if (hydrated) return children;
  return <div ref={ref}>{/* static HTML shell */}</div>;
}
```

This works particularly well for components like dropdown menus, tooltips, and date pickers — complex JS-heavy widgets that most users never open on a given page visit.

> **Check yourself:** A component using interaction-triggered hydration handles a click event. The user clicks it before hydration completes. What does the user experience? How would you handle this?

## Relationship to React 18 and Concurrent Features

React 18's Concurrent Mode features provide progressive hydration as a first-class feature via Selective Hydration (described in the Streaming SSR topic). When React renders server-rendered HTML that contains multiple Suspense boundaries, it hydrates them according to priority:

1. Components the user is interacting with (clicked, hovering) get priority
2. Components in the viewport get hydrated next
3. Off-screen components are deferred

This happens automatically with `createRoot` (React 18's root API). Unlike the manual `LazyHydrate` wrapper above, React manages the scheduling using its scheduler and priority system.

```javascript
// React 18 — Selective Hydration is automatic
const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<App />);
// React will prioritize hydrating components the user interacts with
```

The key difference from the manual approach: React 18 doesn't skip hydrating components — it still hydrates everything, but it uses cooperative scheduling (via the React Scheduler) to yield to user input and prioritize interactive components, rather than doing all hydration in a single blocking pass.

## Progressive Rehydration vs. Partial Hydration

These are related but distinct concepts that often get conflated:

| | Partial Hydration (Islands) | Progressive Rehydration |
|---|---|---|
| JS shipped | Only for interactive components | All components (normal SSR bundle) |
| JS executed | Only for interactive components | For all components, but sequentially |
| Main thread impact | Lower total work | Same total work, better distributed |
| State model | Islands are isolated | Unified component tree |
| Tools | Astro, RSC | Manual implementation or React 18 |

Partial hydration reduces *how much* JS runs. Progressive rehydration controls *when* JS runs. They address the same performance symptom (slow TTI) from different angles and can be combined.

## Gotchas

**Hydration mismatch risk increases.** When you skip or delay hydration of certain parts of the tree, you need to be careful that the static HTML React sees when it eventually hydrates matches what the server rendered. State changes, animations, or dynamic updates that happen to the DOM between SSR and delayed hydration can cause mismatches.

**Interaction-triggered hydration has a visible latency.** When a user clicks a component that hasn't hydrated yet, you need to: (1) start hydration immediately, (2) show some loading indicator, (3) complete the interaction after hydration. If the user is clicking a modal trigger and the modal JS hasn't loaded yet, there's a brief moment where nothing happens. This feels broken. Design for this: either show a spinner in the clickable element or queue the interaction to replay after hydration.

**The setup complexity of manual progressive hydration.** Rolling your own `LazyHydrate` wrapper means you need to handle SSR/browser environment differences, prevent hydration mismatches from the `suppressHydrationWarning` hack, and manage the transition correctly. Libraries like `react-lazy-hydration` exist to address this, and React 18 makes most of it automatic.

**Not a replacement for reducing bundle size.** Progressive rehydration distributes the hydration work over time but the full JS bundle still has to download. If the bottleneck is network (downloading 800KB of JavaScript on a 3G connection), delaying when it executes doesn't help much — you still pay the download cost. The real solution in that case is code splitting + lazy loading, not just scheduling.

## Interview Questions

**Q (High): What is progressive rehydration and how does it improve Time to Interactive?**

Answer: Progressive rehydration is the technique of hydrating SSR content in priority order rather than all at once. In standard hydration, when the JS bundle loads, React hydrates the entire component tree synchronously — this can block the main thread for hundreds of milliseconds on a complex page, during which no user interaction is possible. Progressive rehydration improves TTI by hydrating above-fold, visible, or user-interacted components first, and deferring off-screen components to idle time or viewport entry. The user's above-fold interactions become responsive much sooner because the main thread isn't occupied hydrating the footer and off-screen widgets simultaneously. TTI for above-fold content can drop from 400ms to 50ms even though the total JavaScript to execute is unchanged.

The trap: Confusing progressive rehydration with partial hydration. Partial hydration reduces total JS; progressive rehydration changes the scheduling of how existing JS executes.

---

**Q (High): How does React 18's Selective Hydration differ from manual progressive rehydration?**

Answer: Manual progressive rehydration requires wrapping components in custom wrappers (IntersectionObserver-based, idle-callback-based) that conditionally trigger hydration and manage the static/hydrated state transition. It's a developer-implemented heuristic. React 18's Selective Hydration is a framework-level feature using React's scheduler: when components are wrapped in Suspense boundaries and React detects user interaction (click, hover, focus) on a component that isn't yet hydrated, it immediately prioritizes hydrating that component's Suspense boundary, even interrupting hydration of other parts of the tree. It uses React's priority model (discrete input events get higher priority than deferred work), so the user's clicks are always processed before background hydration work. The practical difference: React 18 handles the prioritization automatically based on actual user behavior; manual approaches use static heuristics (viewport, idle time) that approximate but don't perfectly track user intent.

The trap: Not knowing React 18 handles this automatically via `createRoot`. Describing only the manual approach gives an outdated picture.

---

**Q (Medium): What's the user experience risk of interaction-triggered hydration, and how do you mitigate it?**

Answer: The risk: a user clicks a button that hasn't been hydrated yet. The click fires, but React isn't attached — the event handler doesn't run. To the user, the button appears broken. Mitigation strategies: (1) Queue the interaction — capture the click event before hydration, begin hydration, then replay the queued event after hydration completes; this requires careful event handling and is complex to implement correctly. (2) Show an optimistic loading indicator on the clickable element immediately on click — before hydration finishes, the button shows a spinner, so the user knows the click registered even if the action is slightly delayed. (3) Limit interaction-triggered hydration to components where the hydration is fast (already parsed, just awaiting execution) so the gap is imperceptible. (4) Use `client:idle` (Astro) or schedule hydration to complete before typical user engagement with that component.

The trap: Not acknowledging the "click goes nowhere" problem. This is a real UX failure mode that comes up in senior interviews about progressive hydration.

---

**Q (Medium): When would you choose progressive rehydration over island architecture?**

Answer: The choice hinges on whether state needs to flow between interactive components. Island architecture isolates each interactive component — React context and state don't cross island boundaries. Progressive rehydration keeps the full component tree intact; state, context, and component composition work normally across the whole page. Choose progressive rehydration when: you have a standard React app with shared state between components, you're using React context extensively, or the effort to refactor to island boundaries isn't justified by the JS savings. Choose islands when: the page is predominantly static with a few isolated interactive widgets, the interactive widgets have no meaningful shared state, and you want zero JS shipped for the static majority.

The trap: Describing both approaches without naming the state-sharing constraint as the deciding factor.

---

**Q (Low): Why does progressive rehydration help less when the performance bottleneck is network speed rather than CPU?**

Answer: Progressive rehydration changes when JavaScript executes — it schedules hydration across time, prioritizing above-fold components. But it doesn't change what JavaScript is downloaded. If the bottleneck is a slow connection (3G, constrained mobile network), the full JS bundle must still download before any hydration can start, regardless of the scheduling strategy. A 400KB JS bundle on a 1Mbps connection takes 3.2 seconds to download — every component waits for that download regardless of hydration priority. In this case, the effective fixes are: code splitting + dynamic imports to reduce the initial bundle, partial hydration to ship less total JS, or better caching (service workers, CDN preloading) to reduce download time on repeat visits. Progressive rehydration is a main-thread scheduling optimization, not a network optimization.

The trap: Claiming progressive rehydration helps equally for all performance problems. The distinction between network-bound and CPU-bound performance issues is a senior-level concern.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain the difference between progressive rehydration (scheduling) and partial hydration (skipping)
- [ ] Can describe three hydration strategies: viewport-based, idle-time, interaction-triggered
- [ ] Can explain what React 18 Selective Hydration does and how it's different from manual approaches
- [ ] Can describe the UX risk of interaction-triggered hydration and how to handle it
- [ ] Can explain why progressive rehydration doesn't help when the bottleneck is network speed
- [ ] Can articulate when to choose progressive rehydration vs. island architecture

---
*Next: Resumability (Qwik model) — the most radical rethinking of the hydration problem: serialize the execution state so hydration doesn't need to "replay" anything at all.*
