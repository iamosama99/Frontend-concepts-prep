# Finite State Machines (XState)

## Quick Reference

| Concept | What it is | Why it matters |
|---------|------------|----------------|
| State | A discrete mode the system is in | Eliminates "impossible states" |
| Event | Something that triggers a transition | Makes causes explicit |
| Transition | A state → state edge, triggered by an event | Documents allowed moves |
| Guard | A condition on a transition | Conditional branching without ad-hoc if-chains |
| Effect (action/service) | Side effect triggered on entry, exit, or transition | Separates what happens from when |

---

## What Is This?

A Finite State Machine (FSM) is a model for a system that can be in exactly one of a finite set of states at any given time, moving between states in response to events. No state is implied — every possible mode is named, every transition is explicit.

In frontend engineering, FSMs are used to model UI behavior that has a lifecycle: data fetching (idle → loading → success/error), form submission, authentication flows, multi-step wizards, drag-and-drop interactions.

```js
// Without FSM — boolean soup
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError] = useState(false);
const [isSuccess, setIsSuccess] = useState(false);
const [data, setData] = useState(null);
// Q: can isLoading and isError both be true? Can isSuccess and isError both be true?
// A: yes, unintentionally. Nothing prevents it.

// With FSM — impossible states are impossible by construction
const [state, send] = useMachine(fetchMachine);
// state.value is one of: 'idle' | 'loading' | 'success' | 'error'
// Never two at once. Never neither (there's always an active state).
```

> **Check yourself:** Name a UI component you've built with booleans like `isLoading`, `isOpen`, `isSubmitted`. What invalid combinations did those booleans allow? What bugs could that cause?

---

## Why Does It Exist?

The "boolean explosion" problem is real in production code. Consider a fetch lifecycle modeled with booleans:

```js
const [isLoading, setIsLoading] = useState(false);
const [hasError, setHasError] = useState(false);
const [data, setData] = useState(null);
```

These booleans can independently reach states that should be impossible:
- `isLoading: true, hasError: true` — loading and errored simultaneously?
- `isLoading: false, hasError: false, data: null` — did we fetch yet? Was there no data?
- `isLoading: true, data: {...}` — loading but we already have data?

With 3 booleans, there are 8 possible combinations. Only 4 of them are valid. The other 4 are bugs waiting to happen. With each added boolean, the impossible-state surface doubles.

FSMs eliminate this by making states mutually exclusive: the machine is in exactly one state at all times. The valid states are the model; invalid combinations don't exist.

**Statecharts** (the formalism XState uses) extend basic FSMs with:
- **Hierarchical states** (nested states)
- **Parallel states** (multiple active regions)
- **History** (resuming a previous state)

David Harel invented statecharts in 1987 specifically to handle the complexity that flat FSMs can't express cleanly.

---

## How It Works

### A Basic XState Machine

```js
import { createMachine, assign } from 'xstate';
import { useMachine } from '@xstate/react';

const fetchMachine = createMachine({
  id: 'fetch',
  initial: 'idle',
  context: {
    data: null,
    error: null,
  },
  states: {
    idle: {
      on: {
        FETCH: 'loading', // event 'FETCH' → transition to 'loading'
      },
    },
    loading: {
      invoke: {
        src: 'fetchData', // service (async function)
        onDone: {
          target: 'success',
          actions: assign({ data: (ctx, event) => event.data }),
        },
        onError: {
          target: 'failure',
          actions: assign({ error: (ctx, event) => event.data }),
        },
      },
    },
    success: {
      on: {
        FETCH: 'loading', // allow refetch
        RESET: 'idle',
      },
    },
    failure: {
      on: {
        RETRY: 'loading',
        RESET: 'idle',
      },
    },
  },
}, {
  services: {
    fetchData: () => fetch('/api/data').then(r => r.json()),
  },
});

function DataLoader() {
  const [state, send] = useMachine(fetchMachine);

  return (
    <div>
      {state.matches('idle') && <button onClick={() => send('FETCH')}>Load</button>}
      {state.matches('loading') && <Spinner />}
      {state.matches('success') && <Data value={state.context.data} />}
      {state.matches('failure') && (
        <>
          <Error message={state.context.error?.message} />
          <button onClick={() => send('RETRY')}>Retry</button>
        </>
      )}
    </div>
  );
}
```

### Guards (Conditional Transitions)

Guards are predicates that gate a transition. The transition only fires if the guard returns true:

```js
const authMachine = createMachine({
  // ...
  states: {
    loggedOut: {
      on: {
        LOGIN: {
          target: 'loggedIn',
          cond: 'isValidCredentials', // guard
        },
        LOGIN: {
          target: 'error',
          cond: 'isInvalidCredentials',
        },
      },
    },
  },
}, {
  guards: {
    isValidCredentials: (context, event) => event.password.length >= 8,
    isInvalidCredentials: (context, event) => event.password.length < 8,
  },
});
```

Guards make conditional logic explicit in the state diagram, not buried in component code.

### Hierarchical (Nested) States

Some states have internal structure. A `loggedIn` state might have sub-states: `browsing`, `editing`, `checkingOut`. Instead of making these peers of `loggedOut`, you nest them:

```js
const appMachine = createMachine({
  initial: 'loggedOut',
  states: {
    loggedOut: {
      on: { LOGIN: 'loggedIn' },
    },
    loggedIn: {
      initial: 'browsing', // nested initial state
      states: {
        browsing: { on: { START_EDIT: 'editing' } },
        editing: { on: { SAVE: 'browsing', CANCEL: 'browsing' } },
        checkingOut: { on: { COMPLETE: '#app.loggedOut' } }, // cross-state ref
      },
      on: {
        LOGOUT: 'loggedOut', // works from any sub-state of loggedIn
      },
    },
  },
});
```

The `LOGOUT` event is defined on `loggedIn` and fires from any of its sub-states — you don't have to repeat it in `browsing`, `editing`, and `checkingOut`.

### Parallel States

Sometimes a UI has independent regions that are both active simultaneously. Instead of combining all combinations into a flat set of states, you model them in parallel:

```js
const uiMachine = createMachine({
  type: 'parallel', // both regions are active at all times
  states: {
    sidebar: {
      initial: 'collapsed',
      states: {
        collapsed: { on: { EXPAND: 'expanded' } },
        expanded: { on: { COLLAPSE: 'collapsed' } },
      },
    },
    theme: {
      initial: 'light',
      states: {
        light: { on: { TOGGLE: 'dark' } },
        dark: { on: { TOGGLE: 'light' } },
      },
    },
  },
});

// Active state: { sidebar: 'expanded', theme: 'dark' }
```

Without parallel states, you'd need to enumerate `collapsed.light`, `collapsed.dark`, `expanded.light`, `expanded.dark` — state explosion.

> **Check yourself:** Why are parallel states useful? What problem would they create if you modeled them as a flat machine instead?

---

## XState v5 (Actor Model)

XState v5 embraces the Actor Model: every machine is an actor that communicates with other actors via messages (events). Actors can spawn child actors, and actors can communicate:

```js
import { createActor } from 'xstate';

const actor = createActor(fetchMachine);
actor.start();

actor.subscribe(snapshot => {
  console.log(snapshot.value); // 'idle', 'loading', etc.
});

actor.send({ type: 'FETCH' });
```

This aligns XState with how Erlang/Akka think about distributed systems — each actor is an isolated, self-contained process that only changes state in response to messages.

---

## When to Use XState

Use FSMs when:
- A component has a lifecycle with distinct named phases
- Invalid state combinations need to be prevented by design
- The behavior changes significantly based on the current mode (rendering different UI, allowing different actions)
- You need to visualize or document the state transitions

Don't use XState when:
- State is just a value with no lifecycle (a counter, a toggle, a form field)
- The states are all in the same component and it's simple enough that booleans are readable
- You need the team to learn XState — it has a learning curve that isn't worth it for trivial UI

---

## Gotchas

**1. Context is not state.** In XState, "context" (extended state) is data that changes *within* a state — like a loading progress percentage or accumulated form values. The machine's `state` is the discrete mode. Putting things in context that should determine state leads to guards-as-pseudo-states, which defeats the purpose.

**2. Events are not state.** It's tempting to model "the user clicked submit" as a state (`submitted`). It's not — it's an event that triggers a transition. States are modes you *remain in* for a period of time; events are instantaneous triggers.

**3. Over-engineering simple UI.** A boolean toggle does not need an XState machine. The overhead (serialization, actor lifecycle, devtools) is not worth it for a dropdown open/close.

**4. Missing initial state.** Every machine and every parallel region needs an explicit `initial` state. Forgetting it causes the machine to start in an undefined mode.

**5. Spawning actors without cleanup.** Actors that are spawned but never stopped leak memory and continue processing events. In React, stop actors in cleanup effects.

---

## Interview Questions

**Q (High): What problem do Finite State Machines solve in UI development? Give a concrete example.**

Answer: FSMs eliminate "impossible states" — combinations of booleans or flags that can technically occur but make no semantic sense and cause bugs. The classic example is a data fetching lifecycle: with `isLoading`, `isError`, `data`, you can end up with `isLoading: true, isError: true` simultaneously, or `isLoading: false, isError: false, data: null` which is ambiguous (never fetched? Fetched with no data?). With a FSM, the machine is in exactly one of `idle | loading | success | failure`. Those are the only four possibilities. No code path can set the machine to two states simultaneously. You can also ask: "can I go from success back to loading?" — and the answer is in the machine definition, not scattered across useEffect and event handlers.

The trap: Treating FSMs as just "fancy switch statements." The key insight is mutual exclusivity of states and explicit, auditable transitions.

**Q (High): What is the difference between state and context (extended state) in XState?**

Answer: State (the machine's `state.value`) is the discrete mode the machine is in — one of a named set of nodes (`idle`, `loading`, `success`). It determines which events are valid and which UI branch to render. Context (extended state, `state.context`) is additional data that flows through the machine — like the fetched data, error message, or form values. The distinction matters because context can take on infinite values (any string, any object), while state is bounded (finite number of modes). Putting things in context that should be state — like using `context.phase === 'step2'` to drive branching — defeats the purpose of the machine and recreates the boolean-soup problem the machine was meant to solve.

The trap: Conflating "state" and "context." Interviewers who know XState will probe this distinction.

**Q (Medium): What are statecharts (as opposed to basic FSMs), and why are they more useful for complex UI?**

Answer: Statecharts, formalized by David Harel in 1987, extend flat FSMs with three key features: (1) Hierarchical states — substates nested inside parent states, so a `LOGOUT` event can fire from any substate of `loggedIn` without repeating it; (2) Parallel states — multiple independent regions active simultaneously, avoiding the combinatorial explosion of enumerating every combination; (3) History — the ability to return to the last active substate of a region after leaving it. A multi-step checkout flow with an independent notification system would require dozens of states in a flat FSM; with statecharts, the checkout steps and notification states are separate parallel regions. XState implements statecharts, not just FSMs.

The trap: Saying "XState is just a state machine." The statechart features (hierarchy, parallel, history) are what make it practical for real UI.

**Q (Medium): How does XState's actor model work, and how does it differ from component-local state?**

Answer: In XState v5, every machine instance is an actor — an isolated entity with its own internal state that communicates with other actors exclusively through events (messages). Actors can spawn child actors, communicate with siblings, and respond to events asynchronously. This is the same model as Erlang/Akka actors. Component-local state (useState) is tightly coupled to a single component's render lifecycle — it's destroyed when the component unmounts. An XState actor can outlive any individual component, be shared across the tree, and its state is serializable and inspectable via devtools. The actor model is the right fit when behavior needs to be independent of UI lifecycle or when multiple UI components need to communicate through a shared behavioral model.

The trap: Not knowing the actor model concept. XState v5 leaned fully into this, and interviewers at companies that use XState will ask about it.

**Q (Low): When would you NOT use XState in a production codebase?**

Answer: When the state is simple enough that booleans or a small enum are readable without being error-prone. A sidebar that's either open or closed has two states and one event — adding XState adds indirection with no benefit. Also avoid it when the team isn't familiar with it: XState has a non-trivial learning curve and introducing it requires investment in documentation and onboarding. The right heuristic: if you find yourself drawing state transition diagrams to debug component behavior, that's a sign the component would benefit from a machine. If you can hold the entire state space in your head without a diagram, useState is probably fine.

The trap: Overselling XState. Teams that over-apply it end up with machines for every button click. The answer should show judgment, not fandom.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at this file.

- [ ] Can explain the "impossible states" problem and how FSMs solve it with a concrete example
- [ ] Can write a basic XState machine with at least 3 states, transitions, and an async service
- [ ] Can explain the difference between machine state and context (extended state)
- [ ] Can explain what statecharts add over flat FSMs (hierarchy, parallel, history)
- [ ] Can identify a UI scenario where XState is appropriate vs. overkill

---
*Next: Immutable Data Structures — understanding immutability is what makes FSMs and unidirectional stores predictable: you always know what changed, because the old value is never modified in place.*
