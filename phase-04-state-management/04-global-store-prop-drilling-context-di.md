# Global Store vs. Prop Drilling vs. Context / DI

## Quick Reference

| Pattern | State lives | Update cost | Best for |
|---------|-------------|-------------|----------|
| Prop drilling | Parent component | Low (local re-render) | Shallow trees, 1–2 levels |
| Context | Provider component | All consumers re-render | Low-freq shared state: theme, auth |
| Global store | External store | Only subscribed components | High-freq or complex shared state |
| DI (Angular) | Service class | Angular's CD tree | Cross-cutting services, Angular apps |

---

## What Is This?

When multiple components need the same state, you have to decide where that state lives and how it gets to the components that need it. The options form a spectrum from "pass it down manually" to "broadcast it globally."

This is one of the most consequential architectural decisions in a React application. Choosing wrong leads to either unmaintainable prop chains or unnecessary re-renders.

> **Check yourself:** Name one concrete symptom that tells you prop drilling has become a problem in a codebase. What does it look like in the code?

---

## Why Does It Exist?

React's component model is hierarchical — components compose into trees. State in React is local to a component by default. When two sibling components need the same state, you "lift" it to their common ancestor. When that ancestor is three levels above both consumers, you have a choice: pass it down via props (drilling) or use some form of shared channel (context or store).

Each option was invented to solve the pain of the previous one:

- Prop drilling → works, but becomes unmanageable when intermediate components have to pass props they don't use
- Context → avoids intermediate props, but re-renders all consumers on every value change
- Global store → surgical updates via subscriptions, but adds architectural complexity and indirection

---

## How It Works

### Prop Drilling

The simplest approach: pass state from parent to child as props, down through as many levels as needed.

```js
function App() {
  const [user, setUser] = useState({ name: 'Osama', role: 'admin' });
  return <Dashboard user={user} onUserChange={setUser} />;
}

function Dashboard({ user, onUserChange }) {
  return <Sidebar user={user} onUserChange={onUserChange} />;
  // Dashboard doesn't use user itself — it's just passing it through
}

function Sidebar({ user, onUserChange }) {
  return <UserCard user={user} onUserChange={onUserChange} />;
  // Sidebar doesn't use user either
}

function UserCard({ user, onUserChange }) {
  // Finally, the consumer
  return <div>{user.name}</div>;
}
```

`Dashboard` and `Sidebar` are "prop carriers" — they don't use `user`, they just forward it. This is drilling. Problems:

1. Adding a new prop requires updating every intermediate component
2. Refactoring the component tree breaks the prop chain
3. It's hard to see what a component actually needs vs. what it's just forwarding

When to use it anyway: one or two levels deep. Prop drilling at shallow depth is the simplest and most explicit approach.

### React Context

Context provides a way to share a value anywhere in a subtree without passing it through every intermediate component.

```js
const UserContext = createContext(null);

function App() {
  const [user, setUser] = useState({ name: 'Osama', role: 'admin' });

  return (
    <UserContext.Provider value={{ user, setUser }}>
      <Dashboard />
    </UserContext.Provider>
  );
}

function Dashboard() {
  return <Sidebar />;
  // No longer passes user — it doesn't need to know about it
}

function UserCard() {
  const { user } = useContext(UserContext);
  return <div>{user.name}</div>;
}
```

This solves the intermediate-component problem. `Dashboard` and `Sidebar` no longer need to know about `user`.

**The critical caveat: every component that calls `useContext(UserContext)` re-renders when the context value changes — even if the part of the value it uses hasn't changed.**

```js
// If user object is recreated on every App render, all consumers re-render
<UserContext.Provider value={{ user, setUser }}> // ← new object every render

// Fix: memoize the value
const contextValue = useMemo(() => ({ user, setUser }), [user]);
<UserContext.Provider value={contextValue}>
```

This makes Context appropriate for **low-frequency updates**: theme, locale, authentication status, feature flags. Avoid using Context for state that changes on every keystroke or scroll event.

### Splitting Context for Performance

If a context holds multiple values that change at different rates, split them into separate contexts:

```js
// Merged — any change to theme re-renders auth consumers
const AppContext = createContext({ theme, user, setTheme, setUser });

// Split — theme and auth consumers are independent
const ThemeContext = createContext({ theme, setTheme });
const AuthContext = createContext({ user, setUser });
```

### Global Store (Zustand, Redux, Jotai, Recoil)

A global store lives outside the React tree. Components subscribe to exactly the slices of state they care about — a change to unrelated state doesn't cause them to re-render.

```js
// Zustand — minimal store
import { create } from 'zustand';

const useStore = create(set => ({
  user: null,
  theme: 'light',
  setUser: (user) => set({ user }),
  setTheme: (theme) => set({ theme }),
}));

// Component A subscribes to user only
function UserCard() {
  const user = useStore(state => state.user); // re-renders only when user changes
  return <div>{user?.name}</div>;
}

// Component B subscribes to theme only
function ThemeToggle() {
  const [theme, setTheme] = useStore(state => [state.theme, state.setTheme]);
  // UserCard does NOT re-render when theme changes
  return <button onClick={() => setTheme('dark')}>{theme}</button>;
}
```

The selector (`state => state.user`) is how stores prevent unnecessary re-renders. You're subscribing to a derived slice, not the whole store. Zustand uses shallow equality by default; if the selected value hasn't changed, the component doesn't re-render.

**Redux** follows the same pattern but enforces strict unidirectional flow via actions and reducers:

```js
// Redux — more ceremony, more constraints
const userSlice = createSlice({
  name: 'user',
  initialState: null,
  reducers: {
    setUser: (state, action) => action.payload,
  },
});

function UserCard() {
  const user = useSelector(state => state.user);
  const dispatch = useDispatch();
  // all state changes go through dispatch — explicit, serializable, loggable
}
```

Redux's extra ceremony is worth it when: you need time-travel debugging, serializable action logs, or have multiple people writing state mutations that need auditing.

**Jotai** and **Recoil** take an atom-based approach — state is split into small independent atoms rather than one large store, and components subscribe to individual atoms:

```js
// Jotai
import { atom, useAtom } from 'jotai';

const userAtom = atom(null);
const themeAtom = atom('light');

function UserCard() {
  const [user, setUser] = useAtom(userAtom); // subscribes to userAtom only
}
```

### Dependency Injection (Angular)

Angular's DI system is a first-class alternative to prop drilling and context. Services are classes provided to Angular's injector at module, component, or platform scope. Components declare their needs via constructor injection — the framework resolves and provides instances.

```js
// Service
@Injectable({ providedIn: 'root' }) // singleton across the entire app
class UserService {
  private user$ = new BehaviorSubject(null);
  user = this.user$.asObservable();

  setUser(user) { this.user$.next(user); }
}

// Component
@Component({ ... })
class UserCardComponent {
  user$ = this.userService.user;

  constructor(private userService: UserService) {}
}
```

The `providedIn: 'root'` scope makes this a singleton — one instance shared everywhere. You can also provide at component scope for isolated instances. Angular's DI is hierarchical: a component can override a service for its entire subtree.

DI in Angular is the idiomatic pattern for everything that would be a global store or context in React: auth state, HTTP clients, feature flags, logging services.

> **Check yourself:** Context in React causes all consumers to re-render when the value changes. Name two strategies to limit this re-render cost without switching to a global store.

---

## Choosing the Right Pattern

```
How deep is the state being passed?
  1-2 levels → prop drilling (simplest, most explicit)

How frequently does the state change?
  Low frequency (theme, auth, locale) → Context
  High frequency (form input, scroll position, live data) → Global store

How many components consume it?
  Few, in the same subtree → Context
  Many, scattered across the app → Global store

Do you need time-travel, devtools, or serializable actions?
  → Redux
```

---

## Gotchas

**1. Context is not a performance optimization.** It removes prop-drilling but doesn't prevent re-renders. Every `useContext` consumer re-renders when the provider's value changes. Many engineers discover this in production when adding a counter to Context causes the entire app to re-render on every tick.

**2. Putting functions in Context causes re-renders.** `value={{ user, setUser }}` creates a new object on every parent render. Even if `user` hasn't changed, all consumers re-render because the object reference is new. Memoize with `useMemo`.

**3. Global stores don't solve everything.** Putting server state (data from APIs) in Redux is unnecessary complexity when React Query exists. Global stores are for client state that's genuinely shared. Mixing server state and client state in Redux leads to overly complex reducers and manual cache management.

**4. Selector stability in Zustand/Redux.** `useSelector(state => state.users.filter(u => u.active))` creates a new array on every call, causing re-renders even if the underlying data hasn't changed. Either memoize with `createSelector` (Reselect) or use a more stable selector.

**5. Angular DI scope confusion.** Providing a service in both a root module and a lazy-loaded module creates two instances — the components in the lazy module get a different instance than the rest of the app. This breaks the singleton assumption and is a source of subtle state bugs.

---

## Interview Questions

**Q (High): What's wrong with using React Context for frequently-changing state?**

Answer: Every component that calls `useContext(SomeContext)` re-renders whenever the context value changes — even if the specific part of the value the component uses hasn't changed. For low-frequency state (theme, auth), this is fine. For high-frequency state (search input value, scroll position, real-time data), it causes performance problems because every keystroke or scroll event triggers re-renders across the entire consumer subtree. The fix is either a global store with selectors (so components only re-render when their subscribed slice changes) or splitting the context into smaller, more focused contexts so unrelated consumers don't share update cycles.

The trap: "I'd just use useMemo on the context value." That helps with object identity but doesn't prevent re-renders when the underlying state actually changes.

**Q (High): When would you choose Zustand over Redux? Over React Context?**

Answer: Zustand over Redux when: you don't need strict action-based mutation auditing, you don't need Redux devtools time-travel, and you want minimal boilerplate. Zustand allows direct mutation via `set()` and has almost no ceremony — it's essentially React state but stored outside the component tree. Redux is worth its weight when actions need to be serializable (for replay/devtools), when multiple engineers are writing state mutations that need consistent patterns, or when you're integrating with existing Redux infrastructure.

Zustand over Context when: state changes frequently (Context re-renders all consumers; Zustand re-renders only selectively-subscribed components), when many unrelated components subscribe, or when you need fine-grained selector-based subscriptions.

The trap: Defaulting to Redux for everything. A small app with one or two global state slices is better served by Zustand or even Context.

**Q (High): What is prop drilling, when is it acceptable, and when does it become a problem?**

Answer: Prop drilling is passing state through intermediate components that don't use it, just to deliver it to a deeper consumer. It's acceptable at 1–2 levels — the code is explicit, traceable, and has no performance overhead. It becomes a problem when: intermediate components have to be modified every time a new prop is added, when the prop chain is 3+ levels deep and intermediates clearly don't care about the prop, or when the same data needs to reach many unrelated subtrees. The signal to refactor is when you're modifying `Dashboard` to add a `user` prop that `Dashboard` itself never reads.

The trap: Saying prop drilling is always bad. Explicit props are often better than Context for components that are shallow, tested in isolation, or reused across different contexts.

**Q (Medium): How does Angular's DI compare to React's Context as a mechanism for sharing state?**

Answer: Both solve the same problem — sharing values across a component tree without prop drilling — but with different mechanisms. Angular's DI is class-based and injector-resolved: you define a service class, Angular creates and manages its lifecycle, and components declare it as a constructor dependency. The injector hierarchy mirrors the component tree, so services can be scoped to the whole app, a module, or a specific component subtree. React's Context is simpler — a value provided by a JSX provider, consumed by useContext. Angular's DI scales better for cross-cutting concerns (HTTP interceptors, logging, auth) because services can depend on other services and Angular resolves the dependency graph. React Context is better for simple, subtree-scoped value sharing.

The trap: Treating Angular DI as "just like Context." The injector hierarchy, lifecycle management, and service-to-service dependency resolution are meaningfully different.

**Q (Medium): What is a selector in Redux/Zustand and why does selector stability matter?**

Answer: A selector is a function that derives a value from the store state. In Zustand: `useStore(state => state.user)`. In Redux: `useSelector(state => state.user)`. Both libraries compare the returned value to the previous value (shallow equality) to decide whether to re-render. Selector stability matters because if the selector returns a new reference on every call, the shallow comparison always fails and the component re-renders every time the store changes — even for unrelated state. `state => state.users.filter(u => u.active)` returns a new array every call. The fix: memoize with Reselect's `createSelector` (caches the result until inputs change) or use a stable reference in the selector.

The trap: Not knowing that the equality check is shallow. A selector that computes a derived array is a common source of unexpected re-renders.

**Q (Low): What is Jotai, and how does its atom model differ from a single-store approach?**

Answer: Jotai is a primitive state management library for React based on atoms — small, independent pieces of state that components subscribe to individually. Rather than one large store with selectors, you define many small atoms, and components compose them. This avoids the selector complexity of Redux (you subscribe to the atom directly, not a slice of a large object) and the re-render problem of Context (only components subscribed to a changed atom re-render). Derived atoms (computed from other atoms) update only when their dependencies change, similar to signals. The trade-off: no single centralized devtools view of all state, and no serializable action log. Better for component-level reactive state; worse for app-wide auditable state.

The trap: Not knowing how atoms differ from a store. The key is granularity — atoms are independent reactive cells, not slices of a shared object.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at this file.

- [ ] Can explain why Context causes all consumers to re-render and name two mitigations
- [ ] Can choose between prop drilling, Context, and a store for a given scenario with justification
- [ ] Can write a Zustand store with a selector that prevents unnecessary re-renders
- [ ] Can explain why functions in Context cause reference instability and how to fix it
- [ ] Can explain Angular's DI hierarchy and what a `providedIn: 'root'` vs. component-scoped service means

---
*Next: Finite State Machines (XState) — now that we know where state lives and how it flows, we look at how to model the *shape* of state to eliminate impossible state combinations entirely.*
