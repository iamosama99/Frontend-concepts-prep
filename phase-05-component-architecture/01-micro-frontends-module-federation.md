# Micro-Frontends: Module Federation, Iframes, Web Components

## Quick Reference

| Integration approach | Isolation level | Shared dependencies | Best for |
|---|---|---|---|
| Module Federation | JS context shared | Configurable singletons | Seamless UX, shared design system |
| Iframes | Full (separate browsing context) | None | Strict isolation, legacy embedding |
| Web Components | DOM boundary only | Page-level JS shared | Framework-agnostic component sharing |
| npm package | None (build-time) | Yes | Libraries, not independently deployed apps |

---

## What Is This?

A micro-frontend is an architectural style where an independently deployable unit of frontend code owns a vertical slice of a product — its own UI, logic, and often its own deployment pipeline. Multiple micro-frontends are composed at runtime (or build time) by a shell application to form a single user-facing product.

The idea directly mirrors microservices on the backend: instead of one monolithic React app owned by every team, each team owns and ships its own frontend slice. A checkout team owns `/checkout`, a catalog team owns `/search` and `/product`, and a shell team owns the nav, auth guard, and composition logic.

> **Check yourself:** What is the difference between a micro-frontend and a shared npm package? Which one can a team ship independently without coordinating a release with other teams?

---

## Why Does It Exist?

### Conway's Law

Conway's Law states: "Organizations design systems that mirror their own communication structure." A single frontend codebase owned by five teams will develop the communication overhead of five teams negotiating every merge, shared dependency update, and release schedule.

Micro-frontends are a structural answer to Conway's Law. By aligning code ownership with team boundaries, you let teams move at their own pace. A payments team does not wait for a marketing team's A/B test to clear before shipping a critical checkout fix.

### The coordination tax at scale

In a large SPA monorepo:
- Every team's work shares one CI pipeline — a slow test suite blocks everyone.
- A breaking change in a shared component requires coordinating every consuming team.
- Deploy frequency for any one team is limited by the slowest team.
- Bundle size grows unbounded because all features ship together.

Micro-frontends trade this coordination overhead for operational overhead. That trade is only worth making past a certain scale.

---

## How It Works

### Integration strategies

There are three meaningful runtime integration mechanisms. Build-time integration via npm packages does not count as micro-frontends because it does not allow independent deployment.

---

### Module Federation (Webpack 5)

Module Federation is a Webpack 5 feature that allows one JavaScript bundle (a **remote**) to expose modules that another bundle (a **host**) can load at runtime, without those modules being bundled at build time.

```js
// remote app — webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'checkout',                     // global name on window
      filename: 'remoteEntry.js',           // the manifest file the host fetches
      exposes: {
        './CheckoutApp': './src/CheckoutApp', // what this remote exports
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
      },
    }),
  ],
};
```

```js
// host (shell) app — webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        checkout: 'checkout@https://checkout.example.com/remoteEntry.js',
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
      },
    }),
  ],
};
```

```js
// host app — consuming the remote at runtime
import React, { Suspense, lazy } from 'react';

// This import is resolved at runtime — not bundled at build time
const CheckoutApp = lazy(() => import('checkout/CheckoutApp'));

function Shell() {
  return (
    <Suspense fallback={<div>Loading checkout...</div>}>
      <CheckoutApp />
    </Suspense>
  );
}
```

**How the loading works step by step:**

1. Host bundle executes. It knows about remotes by name but has not fetched them.
2. When `import('checkout/CheckoutApp')` is reached, Webpack runtime fetches `remoteEntry.js` from the checkout domain.
3. `remoteEntry.js` is a small manifest + runtime that registers the remote's chunks.
4. Webpack negotiates shared dependencies: if the host already loaded React 18, the remote uses that same instance (singleton). If versions are incompatible, each side loads its own copy — which can break hooks if both `react` instances are active simultaneously.
5. The actual component chunk is fetched on demand.

**Shared dependencies** are the most critical config decision. Mark `react` and `react-dom` as singletons to prevent the dual React instance bug (hooks rely on a module-level context; two React instances means hooks silently fail or throw).

> **Check yourself:** If a remote requires React 18.2 and the host has React 18.0 loaded as a singleton, what happens? Does Module Federation use the host's version, throw, or load a second copy?

---

### Iframes

An iframe creates a completely separate browsing context. The embedded page has its own JavaScript engine event loop, its own DOM, its own `window`, its own cookies (scoped to its origin), and cannot be accessed by the parent if origins differ.

```html
<!-- Shell embeds checkout as an iframe -->
<iframe
  src="https://checkout.example.com/cart"
  title="Checkout"
  allow="payment"
></iframe>
```

Communication across the iframe boundary uses `postMessage`:

```js
// Shell → iframe
const frame = document.querySelector('iframe');
frame.contentWindow.postMessage(
  { type: 'SET_USER', payload: { userId: '123' } },
  'https://checkout.example.com'   // target origin — always specify this, never '*'
);

// Inside the iframe
window.addEventListener('message', (event) => {
  if (event.origin !== 'https://shell.example.com') return; // validate origin
  if (event.data.type === 'SET_USER') {
    store.dispatch(setUser(event.data.payload));
  }
});
```

**Why iframes still have real use cases:**

- True security isolation — a third-party widget (payment form, chat, ad) runs in a separate origin and cannot read the host page's DOM or cookies.
- Legacy system embedding — an old jQuery application that cannot be rewritten can be embedded without touching its code.
- Browser security model enforces the boundary; no amount of programming error in the iframe can compromise the host page's origin.

**The UX costs are real:**

- Iframes do not share fonts, CSS custom properties, or focus management with the host.
- Scrolling, modals, and tooltips that overflow the iframe boundary require explicit coordination.
- Full-page iframes can cause double scrollbars and break mobile viewport behavior.
- The iframe's URL is not reflected in the parent's address bar, breaking deep linking.

---

### Web Components as integration boundary

Web Components (Custom Elements + Shadow DOM) let any framework render a standard HTML element. The host page inserts `<checkout-app user-id="123"></checkout-app>` and does not care whether the implementation is React, Vue, or plain DOM.

```js
// checkout team ships a Web Component wrapper around their React app
class CheckoutApp extends HTMLElement {
  connectedCallback() {
    const container = document.createElement('div');
    this.appendChild(container);

    // React renders into the shadow DOM (or light DOM)
    ReactDOM.createRoot(container).render(
      <CartRoot userId={this.getAttribute('user-id')} />
    );
  }

  disconnectedCallback() {
    // cleanup: unmount React, remove listeners
  }

  // Observe attribute changes
  static get observedAttributes() { return ['user-id']; }
  attributeChangedCallback(name, oldVal, newVal) {
    if (name === 'user-id') this.updateUserId(newVal);
  }
}

customElements.define('checkout-app', CheckoutApp);
```

```html
<!-- Host, regardless of framework -->
<checkout-app user-id="42"></checkout-app>
```

Shadow DOM provides style encapsulation. CSS inside the shadow tree does not leak out, and host page CSS does not bleed in (unless you use CSS custom properties, which do pierce the shadow boundary — this is intentional and useful for theming).

**The limitation:** Web Components solve style isolation and the integration API surface, but JavaScript still runs in the same page context. Two copies of React on the same page with the same version is fine, but if they are different major versions, you have a bundle size problem. Web Components do not solve dependency sharing the way Module Federation does.

---

## Routing Strategy

A micro-frontend application needs to answer: who owns the URL?

**Shell-level routing** (most common): the shell app reads the URL and decides which micro-frontend to mount. Each micro-frontend receives a base path and handles sub-routes internally.

```js
// Shell — coarse-grained routing
function Shell() {
  const path = window.location.pathname;

  if (path.startsWith('/checkout')) return <CheckoutApp basePath="/checkout" />;
  if (path.startsWith('/catalog')) return <CatalogApp basePath="/catalog" />;
  return <HomeApp />;
}
```

Each micro-frontend then uses its own router instance scoped to its base path. React Router v6:

```js
// Inside CheckoutApp
function CheckoutApp({ basePath }) {
  return (
    <BrowserRouter basename={basePath}>
      <Routes>
        <Route path="/cart" element={<Cart />} />
        <Route path="/payment" element={<Payment />} />
      </Routes>
    </BrowserRouter>
  );
}
```

**The history sync problem:** If both the shell and a micro-frontend create a `BrowserRouter`, they create separate history instances. Navigating inside the micro-frontend will update the browser URL but the shell's router won't know. Solutions:

- Share a single history object via a custom event or a prop.
- Use hash routing in micro-frontends (ugly but isolated).
- Use a framework like single-spa that owns the routing layer entirely and delegates to micro-frontends.

---

## Shared State Between Micro-Frontends

Micro-frontends run in the same JavaScript context (for Module Federation and Web Components). They can share state through:

**Custom Events (pub/sub on the window):**

```js
// Micro-frontend A — user logs in
window.dispatchEvent(new CustomEvent('mfe:auth:login', {
  detail: { userId: '123', token: 'abc' },
  bubbles: true,
}));

// Micro-frontend B — listens for auth events
window.addEventListener('mfe:auth:login', (event) => {
  const { userId, token } = event.detail;
  localStore.setState({ userId, token });
});
```

Custom Events are loosely coupled, require no shared module, and work across any integration approach including iframes (via `postMessage`). Namespace your events (`mfe:domain:action`) to avoid collisions.

**Shared stores via Module Federation:**

A dedicated `shell/store` module can be exposed from the shell and imported by remotes. Because it is a singleton (one instance), all micro-frontends share the same store object.

```js
// shell — exposes its store
exposes: {
  './store': './src/globalStore',
}

// checkout remote — imports the shell's store
import { getStore } from 'shell/store';
const store = getStore();
store.subscribe(() => { /* react to global state changes */ });
```

**URL as shared state:** For state that matters for deep linking (filters, selected items, current step), the URL is the right source of truth. Any micro-frontend can read `window.location` or listen to `popstate`. This also works across iframe boundaries via the parent's URL.

---

## Real Costs

Micro-frontends are not free. The decision requires honest accounting:

| Cost | Description |
|---|---|
| Operational overhead | Each micro-frontend needs its own CI pipeline, deployment, and hosting configuration |
| Versioning and compatibility | Remote APIs (what a remote exposes) must be versioned; breaking changes require coordination |
| Debugging across boundaries | Stack traces stop at the module federation boundary; source maps from remotes must be served separately |
| Duplicated dependencies | If shared config is wrong, React loads twice — larger bundle, broken hooks |
| UX consistency | Each team ships their own design system version unless there is a shared component package |
| Integration testing | End-to-end tests must compose multiple independently running services |
| Developer experience | Local development requires running multiple dev servers and a composition layer |

---

## When Micro-Frontends Are Worth It

**Indicators that micro-frontends solve a real problem:**

- Multiple teams (5+) blocked by a shared codebase's merge queue, test suite, or release schedule.
- Parts of the product have fundamentally different technology requirements (a legacy app embedded alongside a greenfield one).
- Different domains need independent deployment cadence — payments ships 10x per day, marketing ships once a week.
- Organizational chart maps cleanly to product domains that have minimal UI overlap.

**Indicators that micro-frontends are premature:**

- One team owns the entire frontend.
- The main pain is "big codebase" — a well-structured monorepo with good module boundaries solves this without the operational overhead.
- Shared state and shared UI are the dominant patterns — the composition overhead will exceed the coordination savings.
- The team does not yet have strong CI/CD and monitoring practices — micro-frontends multiply these requirements.

A single well-modularized SPA is almost always cheaper to operate than micro-frontends for teams under ~50 engineers.

---

## Gotchas

**The dual React instance bug.** If Module Federation shared config omits `singleton: true` for React, or if two remotes have incompatible React versions, each loads its own copy. Hooks use a module-level ref to track the current fiber; two instances means hooks either fail silently or throw "Invalid hook call." Always mark `react` and `react-dom` as singletons with `requiredVersion`.

**`remoteEntry.js` is a hard runtime dependency.** If the checkout remote's CDN goes down, the shell's `import('checkout/CheckoutApp')` throws. The `Suspense` boundary only catches the loading state; you need an Error Boundary around it to degrade gracefully instead of white-screening the entire shell.

**CSS collisions without Shadow DOM.** Module Federation does not isolate CSS. If the checkout team uses `.button { ... }` as a global style, it leaks into the shell and other micro-frontends. Use CSS Modules, scoped class names, or Shadow DOM to contain styles.

**`postMessage` origin validation is mandatory.** An iframe that listens on `message` without validating `event.origin` accepts messages from any page that embeds it. Always check the origin against an allowlist before acting on any message payload.

**History desync.** Two `BrowserRouter` instances on the same page — one in the shell, one in a micro-frontend — will both listen to `popstate` but they maintain separate internal state. A push from one won't notify the other. Either share a history object or use a single-spa style orchestrator.

**Shared store via Module Federation requires the remote to load before accessing the store.** Dynamic imports are async. If micro-frontend B tries to import from `shell/store` before the shell has initialized, the module resolves to an uninitialized state. Initialize shared stores before mounting remotes.

---

## Interview Questions

**Q (High): What problem do micro-frontends solve, and what problem do they introduce that monoliths don't have?**
Answer: Micro-frontends solve the team coordination problem that emerges when multiple product teams share a single codebase — merge contention, shared CI bottlenecks, and release coupling. They introduce operational overhead (multiple CI/CD pipelines), runtime composition complexity, potential for duplicated dependencies, harder debugging across bundle boundaries, and UX consistency challenges when teams diverge on component versions. The trade-off is worth it when team independence value exceeds operational cost, typically past ~5 teams with clearly separated domains.
The trap: Weaker candidates focus only on the "independence" benefit without honestly weighing the operational and debugging costs. Strong candidates can articulate when the break-even point is and reference Conway's Law as the organizational root cause.

**Q (High): Explain how Module Federation's shared dependency mechanism works and what goes wrong if you configure it incorrectly.**
Answer: Webpack 5's ModuleFederationPlugin lets remotes declare which modules they expose and which dependencies they want to share with the host. At runtime, when a remote is loaded, Webpack negotiates which version of each shared dependency to use. With `singleton: true`, only one instance is ever active — the host's version wins unless it falls outside the remote's `requiredVersion` range, in which case each loads its own. The critical failure mode is React: React hooks use module-level state to track the current rendering context. If two React instances exist on the same page, hooks throw "Invalid hook call" or silently use the wrong context. This happens when `singleton: true` is omitted or when version ranges are incompatible. The fix is always marking React and React-DOM as singletons and aligning version ranges across all remotes.
The trap: Candidates who only know "add shared: { react }" without understanding why singleton matters or what breaks without it.

**Q (High): How do micro-frontends share state, and which mechanism is most appropriate in different scenarios?**
Answer: Three main mechanisms. Custom Events on `window` are loosely coupled, work without shared modules, and are appropriate for cross-cutting events like auth changes or cart updates — use namespaced event names to prevent collisions. A shared store exposed via Module Federation as a singleton is appropriate when multiple micro-frontends need to read/write the same structured state synchronously. URL state is appropriate for anything that must survive refresh, be shareable, or support back navigation — filters, wizard step, selected entity IDs. For iframes, only `postMessage` with strict origin validation works because the JavaScript contexts are separate. The wrong choice is putting shared state in a micro-frontend's internal store and then trying to sync it manually — this creates coupling that defeats the purpose of the split.
The trap: Answering only "custom events" without discussing the URL as shared state or the Module Federation singleton store pattern.

**Q (Medium): What are the UX trade-offs of iframes as a micro-frontend integration strategy?**
Answer: Iframes provide the strongest isolation — separate browsing context, separate JavaScript heap, separate cookies, cross-origin security enforced by the browser. The UX trade-offs: iframes do not inherit host page fonts or CSS custom properties (without explicit configuration), focus management does not flow naturally across the boundary (keyboard navigation, screen readers), overflow content (modals, tooltips, dropdowns) is clipped to the iframe bounds, the iframe URL is not in the browser's address bar so deep linking is broken, and scroll behavior can produce double scrollbars on mobile. Iframes are appropriate when isolation is the primary requirement — third-party payment forms, untrusted ad widgets, or legacy systems that cannot be refactored. For owned micro-frontends where UX coherence matters, Module Federation or Web Components are better.
The trap: Saying iframes are simply "bad" without explaining the legitimate isolation use cases.

**Q (Medium): How do you handle routing in a micro-frontend architecture where each team owns their own routes?**
Answer: The shell owns the top-level routing decision — it reads the URL and decides which micro-frontend to mount. Each micro-frontend receives a `basePath` and manages its own sub-routes internally using its own router instance. The critical problem is history synchronization: if the shell and a micro-frontend each create a `BrowserRouter`, they have separate history state. A navigation inside the micro-frontend updates the browser URL but the shell's router does not react. Solutions: pass a shared history object from the shell into each micro-frontend as a prop, use single-spa as an orchestration layer that owns all history, or scope micro-frontend routing to hash-based routing (simpler but limits deep linking). The URL remains the canonical shared state that survives page refresh and works across all integration strategies.
The trap: Not knowing that two `BrowserRouter` instances on the same page create history sync problems.

**Q (Medium): What is the failure mode when a remote in Module Federation fails to load, and how do you handle it?**
Answer: `import('checkout/CheckoutApp')` is a dynamic import that fetches `remoteEntry.js` at runtime. If that CDN URL is down or returns a non-200, the import throws a network error. `React.Suspense` only handles the pending state (shows the fallback while loading); it does not catch errors. An uncaught rejection from a failing remote import will propagate up and, without an Error Boundary, crash the entire shell's render tree, white-screening the user. The correct pattern is wrapping each lazily-loaded remote in both a `Suspense` (for loading state) and an `ErrorBoundary` (for failure state). The Error Boundary can render a fallback UI — a link to the standalone micro-frontend, a "service temporarily unavailable" message — while the rest of the shell continues functioning.
The trap: Thinking `Suspense` handles errors. It does not — it only handles the pending promise state.

**Q (Low): How do Web Components compare to Module Federation as an integration boundary?**
Answer: Web Components solve the integration API surface: any framework can insert `<checkout-app>` as a standard HTML element, and the Shadow DOM provides CSS scoping. They do not solve dependency sharing. Both the shell and the checkout Web Component can run their own React instance, which means larger bundles — but they won't conflict at the hook level because each runs in its own component tree without the singleton constraint. Module Federation solves dependency sharing — one React instance, shared at runtime — but requires Webpack on both sides (or Vite with a plugin) and does not provide CSS isolation. In practice, teams often combine them: Module Federation for dependency optimization plus CSS Modules or BEM for style isolation. Pure Web Component integration is good when the micro-frontend team needs true framework independence and isolation is more important than bundle efficiency.
The trap: Thinking Shadow DOM prevents JavaScript conflicts. It does not — it is a DOM/CSS boundary, not a JS boundary.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Explain Conway's Law and why it motivates micro-frontends as an organizational pattern, not just a technical one
- [ ] Walk through how Module Federation's runtime loading works step by step, including how shared dependencies are negotiated
- [ ] Explain what breaks when React is not configured as a singleton in Module Federation and why hooks are the failure point
- [ ] Compare iframes, Module Federation, and Web Components on isolation level, shared dependency handling, and UX trade-offs
- [ ] Describe three mechanisms for sharing state across micro-frontends and which scenario each fits
- [ ] State the team-size and domain-structure conditions under which micro-frontends are worth the operational cost

---
*Next: Monorepo Management vs. Polyrepo — micro-frontends raise the question of how to organize the repositories themselves; monorepo and polyrepo are the two answers with very different trade-offs.*
