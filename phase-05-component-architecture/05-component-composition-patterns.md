# Component Composition Patterns: HOCs, Render Props, Hooks, Compound Components

## Quick Reference

| Pattern | Core mechanism | Primary use today |
|---------|---------------|-------------------|
| Higher-Order Component (HOC) | Function that wraps a component and returns an enhanced component | Wrapping third-party components; class component contexts |
| Render Props | Prop that is a function; parent calls it to delegate rendering | Rare — mostly replaced by hooks |
| Custom Hooks | Function that calls other hooks and returns stateful values/functions | Logic sharing — almost always the right choice |
| Compound Components | Parent + children share implicit state via Context | Complex multi-part UI (Tabs, Accordion, Select) |

---

## What Is This?

Every non-trivial UI has behavior that needs to be shared across components: tracking whether a user is authenticated, logging analytics events, fetching data, managing open/closed toggle state. The question is how to share that behavior without duplicating it and without creating a tangled prop chain.

Component composition patterns are the set of techniques React evolved to solve this. They are not framework features — they are conventions built from ordinary JavaScript that allow stateful logic and rendering to be reused, recombined, and kept separate from each other.

```js
// The problem in a nutshell: three components need "is user authenticated" logic
function Header() {
  const [user, setUser] = useState(null);
  useEffect(() => { fetchUser().then(setUser); }, []);
  // ... use user
}

function Sidebar() {
  const [user, setUser] = useState(null);
  useEffect(() => { fetchUser().then(setUser); }, []);
  // ... same logic, duplicated
}
```

Composition patterns are answers to: how do we extract this and share it cleanly?

> **Check yourself:** Name three distinct categories of reusable logic that composition patterns are meant to extract. (Logic? Rendering? State? All three?)

---

## Why Does It Exist?

### React's Inheritance Dead End

Most object-oriented UI frameworks lean on class inheritance for reuse: a `BaseComponent` holds common logic and subclasses extend it. React deliberately chose against this. Dan Abramov's team found that inheritance hierarchies in UI quickly become tangled — you inherit too much, you override the wrong things, and the relationship between parent and child becomes brittle and implicit.

React's composition model instead says: favor wrapping over inheriting. Components that need shared behavior should receive it, not inherit it. This led directly to HOCs as the first serious composition mechanism.

### The Evolution Arc

The history of composition patterns in React maps almost perfectly to React's broader maturation:

1. **Pre-2016:** No standard pattern. Teams either duplicated logic or invented ad-hoc mixins (which the React team eventually deprecated from class components because they caused name collisions and unclear ownership).
2. **2016–2018:** HOCs became the community standard. Libraries like Redux (`connect`), React Router (`withRouter`), and Relay all used HOCs.
3. **2018:** Render Props emerged as an alternative that addressed HOC's opacity and debugging problems.
4. **Late 2018–present:** Hooks (React 16.8) made both HOCs and Render Props largely obsolete for logic sharing. Custom hooks can do everything both could do, with better ergonomics and none of the wrapping overhead.
5. **Parallel evolution:** Compound Components emerged from component library authors (Reach UI, Headless UI, Radix) who needed multi-part component APIs that felt native — not a single monolithic component with dozens of boolean props.

Understanding this arc is what separates a senior answer from a junior one in an interview.

---

## How It Works

### Higher-Order Components

A HOC is a function that takes a component and returns a new component with additional capabilities injected:

```js
// HOC signature: Component In → Enhanced Component Out
function withAuth(WrappedComponent) {
  return function AuthenticatedComponent(props) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
      fetchCurrentUser()
        .then(setUser)
        .finally(() => setLoading(false));
    }, []);

    if (loading) return <Spinner />;
    if (!user) return <Redirect to="/login" />;

    // Pass through all original props plus the injected `user` prop
    return <WrappedComponent {...props} user={user} />;
  };
}

// Usage
const ProtectedDashboard = withAuth(Dashboard);
// <ProtectedDashboard someOtherProp="value" />
// Dashboard receives: { someOtherProp: "value", user: {...} }
```

The HOC pattern works by exploiting the fact that React components are just functions (or classes). A HOC is a higher-order function — it takes a function and returns a function, same as `Array.prototype.map` takes a function and returns an array.

**Why HOCs cause problems:**

```js
// Prop collision: two HOCs both inject `data` prop
const Enhanced = withUserData(withPostData(Dashboard));
// Which `data` prop does Dashboard receive? The outer HOC's, silently clobbering the inner's.

// Wrapper hell in DevTools
// <withAuth(withAnalytics(withTheme(withLocale(Dashboard))))>
//   <withAnalytics(withTheme(withLocale(Dashboard)))>
//     <withTheme(withLocale(Dashboard))>
//       <withLocale(Dashboard)>
//         <Dashboard>
// DevTools is unreadable. Stack traces are opaque.

// ref forwarding is broken by default
const Enhanced = withAuth(Dashboard);
<Enhanced ref={dashboardRef} /> // ref points to the HOC wrapper, not Dashboard
// Fix requires React.forwardRef in every HOC — error-prone to remember
```

> **Check yourself:** Given `const A = withX(withY(withZ(Comp)))`, what happens if `withX` and `withZ` both inject a prop named `data`? Which one wins, and would you see an error?

---

### Render Props

Render Props inverts the HOC model: instead of wrapping a component, you give control of rendering back to the consumer by accepting a function as a prop:

```js
// The DataFetcher component owns fetching logic, delegates rendering to the caller
function DataFetcher({ url, render }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch(url)
      .then(r => r.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);

  // Call the render prop — consumer decides what to render
  return render({ data, loading, error });
}

// Usage
function UserProfile({ userId }) {
  return (
    <DataFetcher
      url={`/api/users/${userId}`}
      render={({ data, loading, error }) => {
        if (loading) return <Spinner />;
        if (error) return <ErrorMessage error={error} />;
        return <ProfileCard user={data} />;
      }}
    />
  );
}
```

The key improvement over HOCs: you can see exactly what data the behavior provides at the call site. There's no injection mystery — `{ data, loading, error }` is explicit in the render function signature.

**The children-as-function variant** is the same pattern with `children` instead of a named prop:

```js
<DataFetcher url="/api/users">
  {({ data, loading }) => loading ? <Spinner /> : <UserList users={data} />}
</DataFetcher>
```

**Why Render Props still have problems:**

```js
// Callback hell when composing multiple render props
<DataFetcher url="/api/users">
  {({ data: users }) => (
    <AuthProvider>
      {({ user }) => (
        <ThemeProvider>
          {({ theme }) => (
            // Finally get to render something useful
            <UserList users={users} currentUser={user} theme={theme} />
          )}
        </ThemeProvider>
      )}
    </AuthProvider>
  )}
</DataFetcher>
// Every level adds indentation, every level is a new component (performance overhead)
// This is JSX pyramid of doom
```

---

### Custom Hooks — The Superseding Pattern

Hooks extract stateful logic into a plain JavaScript function that can be called directly in any component:

```js
// The same data fetching logic, as a hook
function useDataFetcher(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    fetch(url)
      .then(r => r.json())
      .then(d => { if (!cancelled) setData(d); })
      .catch(e => { if (!cancelled) setError(e); })
      .finally(() => { if (!cancelled) setLoading(false); });
    return () => { cancelled = true; };
  }, [url]);

  return { data, loading, error };
}

// Usage — flat, no nesting, no wrapper components
function UserProfile({ userId }) {
  const { data: user, loading, error } = useDataFetcher(`/api/users/${userId}`);
  const { user: currentUser } = useAuth();
  const { theme } = useTheme();

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <ProfileCard user={user} currentUser={currentUser} theme={theme} />;
}
```

Compare the three approaches side-by-side. The hook version:
- Has no wrapper components adding noise to DevTools
- Has no prop injection — every value is explicitly destructured from the hook call
- Has no naming collision risk — hook return values are locally named at the call site
- Composes horizontally (multiple `useX()` calls) rather than vertically (nested HOCs or nested render props)
- Can itself call other hooks — hooks compose cleanly

**The one thing hooks cannot do:** they cannot render anything. A hook is pure logic — it cannot put JSX on screen. This is precisely why Compound Components still exist.

---

### Compound Components

Compound Components solve a different problem from HOCs/render props/hooks. The problem is not logic sharing — it's API design for multi-part UI.

Consider building a `Tabs` component. The naive approach:

```js
// Naive: a monolith with tons of props
<Tabs
  tabs={[
    { label: 'Profile', content: <ProfilePanel /> },
    { label: 'Settings', content: <SettingsPanel /> },
  ]}
  defaultActiveIndex={0}
  onTabChange={handleChange}
  tabStyle="underline"
  panelPadding={16}
  // ... 15 more props as requirements grow
/>
```

Every new requirement adds another prop. The component becomes a configuration object rather than a composable UI element. The consumer has no control over the structure.

The Compound Component pattern instead exposes the parts as separate components that share implicit state through Context:

```js
// Compound Component API — consumer controls structure
<Tabs defaultIndex={0} onChange={handleChange}>
  <Tabs.List>
    <Tabs.Tab>Profile</Tabs.Tab>
    <Tabs.Tab>Settings</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panels>
    <Tabs.Panel><ProfilePanel /></Tabs.Panel>
    <Tabs.Panel><SettingsPanel /></Tabs.Panel>
  </Tabs.Panels>
</Tabs>
```

The consumer can reorder `Tabs.List` and `Tabs.Panels`, wrap `Tabs.Tab` in a tooltip, add custom markup between tabs — without the `Tabs` component needing to know or support any of it. The parent controls state; the children receive it.

**How it works — the Context implementation:**

```js
const TabsContext = React.createContext(null);

function Tabs({ children, defaultIndex = 0, onChange }) {
  const [activeIndex, setActiveIndex] = useState(defaultIndex);

  const selectTab = (index) => {
    setActiveIndex(index);
    onChange?.(index);
  };

  // Share state implicitly through Context — children never see this as props
  return (
    <TabsContext.Provider value={{ activeIndex, selectTab }}>
      {children}
    </TabsContext.Provider>
  );
}

function TabList({ children }) {
  return <div role="tablist">{children}</div>;
}

function Tab({ children, index }) {
  const { activeIndex, selectTab } = useContext(TabsContext);
  const isActive = activeIndex === index;

  return (
    <button
      role="tab"
      aria-selected={isActive}
      onClick={() => selectTab(index)}
    >
      {children}
    </button>
  );
}

function Panel({ children, index }) {
  const { activeIndex } = useContext(TabsContext);
  if (activeIndex !== index) return null;
  return <div role="tabpanel">{children}</div>;
}

// Attach sub-components as properties of the parent
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = Panel;
```

The `index` prop in this implementation is explicit — it requires the consumer to pass `index={0}`, `index={1}`. A more sophisticated implementation uses `React.Children.map` or a registration pattern to inject indices automatically, so the consumer never has to count:

```js
// Auto-index pattern using React.Children.map + cloneElement
function TabList({ children }) {
  return (
    <div role="tablist">
      {React.Children.map(children, (child, index) =>
        React.cloneElement(child, { index })
      )}
    </div>
  );
}
// Now <Tabs.Tab> without index={n} just works
```

This is the one legitimate use of `cloneElement` — injecting positional metadata that children cannot know themselves. (More on `cloneElement` in File 6.)

> **Check yourself:** Why is Context the right mechanism for Compound Components? Why not just pass props from parent to each child explicitly?

---

## The Evolution Arc — Interview Mental Model

Understanding why these patterns exist in sequence matters more than memorizing them:

| Problem | Pattern that emerged | Why it was superseded |
|---------|---------------------|----------------------|
| Code duplication across class components | HOC | Prop collision, wrapper hell, opaque DevTools, ref issues |
| HOC's opacity — you can't see what gets injected | Render Props | Callback hell, indentation pyramid, runtime overhead |
| Render Props' nesting — horizontal logic stacking is cleaner | Custom Hooks | Can't render — doesn't solve multi-part UI state sharing |
| Multi-part UI components with tangled prop APIs | Compound Components | Still the right answer — no superseding pattern |

HOCs and Render Props are not "wrong" — they were correct solutions to the constraints of their era. HOCs are still the right pattern when wrapping a third-party component that you can't modify, or when working in a class component context. Render Props still appear in some libraries. The key interview insight: hooks didn't beat them by being clever — they beat them by separating two concerns that HOCs and render props conflated: *logic* (hooks) vs. *rendering* (components).

---

## When to Use Each Pattern

**HOC** — use when:
- Wrapping a third-party component you can't modify (e.g., adding error boundaries to a charting library component)
- Working in a class component codebase
- The behavior wraps the entire component including its lifecycle (though hooks can usually do this too)

**Render Props** — use when:
- You encounter them in a library you're consuming
- The "children as function" form is the cleanest API for a particular component (e.g., a `<Transition>` component that needs to pass animation state to its children)
- Extremely rare in greenfield code today

**Custom Hooks** — use when:
- Sharing stateful logic between components (auth, data fetching, subscriptions, media queries, local storage sync)
- This covers the vast majority of logic-sharing cases

**Compound Components** — use when:
- Building a multi-part UI component where the parts must share state (Tabs, Accordion, Dropdown, Select, RadioGroup, Modal with header/body/footer)
- The consumer needs control over the structure and ordering of parts
- A single-component API would require an ever-growing prop surface

---

## Gotchas

**1. HOC display names disappear in DevTools.** By default, a HOC returns an anonymous function. DevTools shows `<Unknown>` or `<Component>`. Fix: set `displayName` explicitly on the returned component: `AuthenticatedComponent.displayName = \`withAuth(\${WrappedComponent.displayName || WrappedComponent.name})\``. Every production HOC should do this.

**2. HOCs break ref forwarding.** Refs attached to a HOC-wrapped component point to the HOC's wrapper, not the inner component. Fix requires `React.forwardRef` in the HOC, which adds boilerplate and is easy to forget. Hooks have no wrapping component, so there's nothing to forward through.

**3. Prop naming collision in stacked HOCs is silent.** If two HOCs inject the same prop name, the outermost HOC's value silently overwrites the inner one. No error, no warning. This is production-level data corruption and the hardest HOC bug to debug.

**4. Context re-renders all consumers when value changes.** In Compound Components, the Context value object should be memoized if the parent re-renders frequently. If `TabsContext.Provider` receives a new object reference on every render (e.g., `value={{ activeIndex, selectTab }}`), every `useContext(TabsContext)` call re-renders even when `activeIndex` didn't change. Fix: `useMemo(() => ({ activeIndex, selectTab }), [activeIndex, selectTab])`.

**5. Custom hooks don't solve Compound Component state sharing.** A hook returns values — it doesn't inject them into children. If you have `<Parent>` and `<Child>` and `<GrandChild>` that all need the same state, a hook in Parent cannot get that state to GrandChild without threading props through Child. Context is the right tool. A custom hook *inside* the Context Provider is fine.

**6. `React.Children.map` breaks with Fragment children.** If the consumer wraps `<Tabs.Tab>` children in a Fragment, `React.Children.map` iterates the Fragment as a single child, not its contents. This is a real edge case in compound component implementations that use `cloneElement` for index injection. It's one reason some libraries use a Context-based registration pattern (children register themselves) instead of `cloneElement`.

---

## Interview Questions

**Q (High): Explain the evolution from HOCs to Render Props to Hooks. What specific problem did each supersede the previous for?**

Answer: HOCs were the first serious mechanism for sharing stateful logic across class components. They worked but caused three practical problems: prop injection is invisible at the call site (you don't know what `withAuth` adds without reading its source), stacking HOCs causes wrapper hell in DevTools making debugging painful, and they break ref forwarding silently. Render Props fixed the opacity problem — the function signature makes explicit what data the behavior provides. But composing multiple render props creates a deeply nested callback pyramid, each level adding both indentation and a new component instantiation. Custom Hooks superseded both by allowing horizontal composition: multiple `useX()` calls in sequence, no nesting, no wrapper components, no injection. Hook return values are locally named at the call site so there's no collision. Hooks eliminated the HOC and render props problems entirely for *logic sharing*. Compound Components were not superseded — they solve a different problem (multi-part UI state sharing) that hooks cannot address, because hooks don't render anything.

The trap: Treating this as purely historical rather than understanding which pattern is still correct today and why. HOCs for wrapping third-party components, hooks for logic sharing, compound components for multi-part UI — these coexist.

**Q (High): How do Compound Components work internally? What makes the children able to access the parent's state without prop drilling?**

Answer: Compound Components use React Context as a private communication channel. The parent component creates a Context, wraps its children in a Provider, and puts its state into the Context value. Each child component calls `useContext` with that same Context. The children can be anywhere in the subtree — direct children, grandchildren, or deeper — and they receive the same state. From the consumer's perspective, they're just writing `<Tabs.Tab>` without passing any props for state — the state subscription happens inside `Tab` via `useContext`, invisibly. The parent-child relationship is implicit through Context, not explicit through props. This is why the API feels like HTML elements (you don't pass `selected` to `<option>` manually — the browser's native select handles it). The Context scoping means multiple `<Tabs>` instances on the same page each have their own Context and their children don't bleed into each other.

The trap: Saying "the parent passes props to children" — that defeats the whole purpose. Or not knowing Context is involved at all and guessing at the mechanism.

**Q (High): When would you still use a HOC over a custom hook in 2024?**

Answer: Three cases. First, wrapping a third-party component you cannot modify — if a charting library exports a component and you need to add an error boundary or analytics tracking around it, a HOC is the natural fit. You can't add a hook to a component you don't own. Second, class component contexts — if you're maintaining a legacy React codebase where a component is a class and can't become a function component, hooks are unavailable. HOCs (which return function components) can inject values into class components via props. Third, some cross-cutting concerns where you need to wrap the component's entire render — though this is now largely achievable with a thin wrapper functional component. For any new code in a hooks-capable context, the hook is almost always cleaner.

The trap: Saying HOCs are never appropriate today. They still have a real niche.

**Q (High): What is the difference between a Compound Component and a component that accepts many configuration props? When is each appropriate?**

Answer: A large-prop component is simpler to implement but creates a rigid API: every capability needs its own prop, the structure is fixed, and adding requirements grows the prop surface. A Compound Component exposes the parts separately, letting the consumer control structure, ordering, and custom content at each injection point. The cost of Compound Components is implementation complexity — you need Context, you need to define the sub-component API, and you need to handle edge cases (cloneElement and Fragments, for instance). The right choice depends on who controls the rendering. If you're building a product component for a single team with known requirements, a config-prop API is fine and simpler. If you're building a component library that ships to many teams with unpredictable requirements, Compound Components give consumers the flexibility they need without requiring the library to anticipate every use case.

The trap: Always recommending Compound Components for "reusability" — they have real complexity costs that aren't worth it for simple, stable components.

**Q (Medium): What prop collision looks like in HOCs and how does it manifest as a bug?**

Answer: If `withUserData` injects `{ data: user }` and `withPostData` also injects `{ data: post }`, and you stack them as `withUserData(withPostData(Component))`, the outer HOC's `data` overwrites the inner's. The component receives only one `data` prop — whichever HOC is outermost. There is no error. The inner HOC's data is silently discarded. This is especially insidious in TypeScript-less codebases because the type system isn't catching the collision. In production, this manifests as components showing the wrong data — the symptoms are a data mismatch that doesn't correspond to any obvious code path. The fix is namespacing injected props (`userProps`, `postProps`) but that's a convention, not an enforced guarantee.

The trap: Not knowing the specific mechanism (outer clobbers inner) and just saying "there might be naming conflicts."

**Q (Medium): Why is `React.Children.map` with `cloneElement` considered a sign of poor API design in most cases?**

Answer: `cloneElement` modifies a React element that was created by the caller — you're mutating someone else's JSX. This creates implicit coupling: the parent knows about the children's internal prop interface, the children rely on props being secretly injected, and the consumer can't easily understand what props a child receives without reading the parent's source. It also breaks with Fragment children and is not composable. The one legitimate use is Compound Components injecting positional index — metadata children cannot know themselves. But for anything else (injecting callbacks, passing theme data, adding className), there are better tools: Context for state, composition via explicit props, or hooks. The smell to watch for: "I need the parent to add `isDisabled` to all children" — that's a signal to reach for Context instead of cloneElement.

The trap: Saying cloneElement is always bad and forgetting the compound component index injection case.

**Q (Medium): In a Compound Component implementation, why should the Context value be memoized?**

Answer: `TabsContext.Provider` receives a `value` prop. When `Tabs` re-renders — even for an unrelated reason — if the value is an inline object like `{ activeIndex, selectTab }`, React creates a new object reference on every render. Every component that calls `useContext(TabsContext)` compares the previous context value to the new one by reference. A new object reference always fails the equality check, so every consumer re-renders even if `activeIndex` didn't change. This defeats the React rendering optimization model. The fix is `const value = useMemo(() => ({ activeIndex, selectTab }), [activeIndex, selectTab])` in the Provider — stable reference when the contents are stable. This is the same root cause as the reference stability problem with `useMemo` from Phase 4: object identity, not deep equality, drives React's reconciliation decisions.

The trap: Not connecting this to the broader reference stability concept. Interviewers want to see you reason from first principles, not just recite the fix.

**Q (Low): Can you implement a Render Props component and then show how a custom hook replaces it?**

Answer:
```js
// Render Props version
function MouseTracker({ render }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  useEffect(() => {
    const handler = (e) => setPosition({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);
  return render(position);
}
// Usage
<MouseTracker render={({ x, y }) => <span>{x}, {y}</span>} />

// Hook version — the logic is identical, no wrapper component
function useMousePosition() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  useEffect(() => {
    const handler = (e) => setPosition({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);
  return position;
}
// Usage — flat, no nesting
function Cursor() {
  const { x, y } = useMousePosition();
  return <span>{x}, {y}</span>;
}
```
The hook version has no wrapper in the component tree, the logic is testable as a plain function, and composing with other hooks is flat.

The trap: Not showing the actual implementation and just describing it abstractly.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at this file.

- [ ] Can explain the HOC → Render Props → Hooks evolution and the specific problem each superseded the previous for
- [ ] Can write a working HOC (`withAuth`) and name its three failure modes (prop collision, wrapper hell, ref forwarding)
- [ ] Can implement a Compound Component (`Tabs`) using Context, including how sub-components are attached as properties of the parent
- [ ] Can explain why custom hooks do not replace Compound Components, and what the structural difference is
- [ ] Can name when a HOC is still the right choice in 2024
- [ ] Can explain why the Context value in a Compound Component should be memoized and what bug occurs when it isn't

---
*Next: Slot-based Composition vs. Children Prop Patterns — compound components are one form of composition; slots and the children prop are the complementary pattern for controlling layout and content injection.*
