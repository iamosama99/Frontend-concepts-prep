# Server State vs. Client State vs. URL State vs. Form State

## Quick Reference

| Type | Owned by | Lives where | Tooling |
|------|----------|-------------|---------|
| Server state | Backend | Remote DB, cached locally | React Query, SWR, Apollo |
| Client state | Frontend | Memory (component/store) | useState, Zustand, Redux |
| URL state | Browser/router | URL bar (searchParams, path) | URLSearchParams, router |
| Form state | User input session | DOM + controlled state | react-hook-form, Formik |

---

## What Is This?

Not all state is the same thing. "State" in frontend development covers at least four distinct categories that have different ownership models, lifetimes, and appropriate tooling. Mixing them up — storing server data in Redux, or putting URL-worthy filters in component state — is one of the most common architectural mistakes in large applications.

Recognizing which category a piece of state belongs to before you write any code is the first and most important decision.

> **Check yourself:** Before reaching for useState or a global store, can you name which of the four categories your state belongs to? What question would you ask to determine this?

---

## Why Does It Exist?

The distinction matters because each category has fundamentally different requirements:

- Server state goes stale. It needs background refetching, cache invalidation, deduplication of parallel requests, and loading/error handling. Managing this by hand with `useState + useEffect` leads to race conditions, waterfall fetches, and missing loading states.
- Client state is ephemeral and synchronous. It doesn't need caching or network logic. Using React Query for "is the modal open?" is overkill.
- URL state needs to survive page refresh, be shareable via link, and support browser back/forward. Storing it in component state means reloading the page loses the user's context.
- Form state is a transient snapshot of user intent. It's neither the server's authoritative data nor stable enough to cache. It has unique needs: validation, dirty tracking, field-level errors, and submission lifecycle.

Treating all state as "useState" or "Redux everything" leads to code that's harder to maintain and has subtle bugs.

---

## How It Works

### Server State

Server state is the canonical data that lives in a database and is delivered to the client via API. The client holds a cached copy that may be stale.

```js
// Without a library — the wrong way
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      });
    // Missing: cleanup, error handling, race conditions, cache, dedup
  }, [userId]);
}

// With React Query — the right way
function UserProfile({ userId }) {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(r => r.json()),
    staleTime: 5 * 60 * 1000, // 5 min before refetch
  });
}
```

React Query handles: deduplication (two components fetching the same key share one request), background refetching on window focus, retries, cache invalidation, optimistic updates, and devtools. SWR does the same with a slightly different API and philosophy (simpler, smaller).

### Client State

Ephemeral UI decisions that don't need to survive a page reload and have no server-side equivalent.

```js
// Appropriate for useState — no need for a global store
function Sidebar() {
  const [isOpen, setIsOpen] = useState(false);
  // ...
}

// Appropriate for a store only if shared across distant components
const useSidebarStore = create(set => ({
  isOpen: false,
  toggle: () => set(state => ({ isOpen: !state.isOpen })),
}));
```

The rule: if only one component (or a tight subtree) needs this state, `useState`. If genuinely shared across distant parts of the tree, a store.

### URL State

State that belongs in the URL because it affects what the page renders and should be linkable.

```js
// Filters, pagination, search query — URL is the source of truth
function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();

  const category = searchParams.get('category') ?? 'all';
  const page = Number(searchParams.get('page') ?? '1');

  function setCategory(cat) {
    setSearchParams(prev => {
      prev.set('category', cat);
      prev.set('page', '1'); // reset page on filter change
      return prev;
    });
  }
}
```

The URL is the canonical source for this state. On mount, read from the URL. On change, write to the URL. Never mirror URL state into `useState` — that creates two sources of truth.

A good heuristic: if the user should be able to copy the URL and send it to someone who arrives on the exact same view, it's URL state.

### Form State

Controlled input during an active editing session. It's temporary until submitted, then either discarded or turned into a server state mutation.

```js
// react-hook-form — avoids re-rendering the whole form on every keystroke
function LoginForm() {
  const { register, handleSubmit, formState: { errors } } = useForm();

  const onSubmit = async (data) => {
    await loginUser(data); // mutates server state
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email', { required: 'Email is required' })} />
      {errors.email && <span>{errors.email.message}</span>}
    </form>
  );
}
```

Form state is distinct from server state even when it mirrors server data (e.g., an "edit profile" form). The edited values live in form state until submit; the authoritative values live in server state. Conflating them creates bugs — especially when the user edits the form and a background refetch overwrites their changes.

> **Check yourself:** A user searches for products with filters, pages to page 3, and shares the URL. The recipient opens it and sees a blank state. What was done wrong, and what's the fix?

---

## Choosing the Right Category

```
Is the data fetched from a server?
  → Yes: Server state. Use React Query or SWR.

Is it something the user might want to share via link or bookmark?
  → Yes: URL state. Use searchParams/router.

Is it an active editing session that will be submitted?
  → Yes: Form state. Use react-hook-form or Formik.

Is it everything else (UI toggles, local preferences, computed UI values)?
  → Client state. useState or a store if shared.
```

The mistake most teams make is defaulting everything to the third or fourth category because it's familiar, then adding complexity when the requirements turn out to belong in the first or second.

---

## Gotchas

**1. Syncing server state into useState creates stale data bugs.** If you fetch a user on mount and store it in `useState`, background changes to that user are invisible until the component remounts. Use React Query's cache directly — don't copy server data into local state.

**2. Storing URL state in useState causes shareability failures.** When you navigate to a page, if the component initializes state from `useState` rather than reading from the URL, every user starts from the same default, not from the URL they were given.

**3. Form state leaking into server state.** Passing a form field's onChange directly to a server mutation on every keystroke burns requests and creates race conditions. Form state should accumulate until submit, then trigger a single mutation.

**4. Over-centralizing client state.** Not every boolean needs to be in Redux. Global stores have an initialization cost (serialization, devtools, rendering overhead). If the state is local to a component, keep it local.

**5. Resetting form state from server data on background refetch.** If you populate a form from server data and then React Query refetches in the background, the controlled form fields will be overwritten while the user is mid-edit. Use `defaultValues` (set once on mount) rather than `values` (reactive).

---

## Interview Questions

**Q (High): What's the difference between server state and client state? Why does this distinction matter architecturally?**

Answer: Server state is remote data that the client caches — it goes stale, needs refetching, and requires cache invalidation. Client state is ephemeral UI state that only the browser owns. The distinction matters because they have completely different lifecycle requirements. Server state needs deduplication, loading/error handling, retries, and background updates. Treating server state as client state (useState + useEffect fetch) leads to race conditions, missing edge cases, and redundant requests. Libraries like React Query exist specifically to manage server state's lifecycle correctly.

The trap: Saying "Redux for everything" or not recognizing the four-category model at all.

**Q (High): When should state live in the URL rather than in React state?**

Answer: State belongs in the URL when: (1) the page should be bookmarkable/shareable in that exact state, (2) the browser back/forward buttons should navigate through it, or (3) a page refresh should restore it. Classic examples: search queries, filters, pagination, selected tab when the tab affects content. The URL is the canonical source of truth — read from it on mount, write to it on change, never mirror it into useState.

The trap: "I'd put it in useState for now and move it to the URL later." That "later" almost never happens, and users learn early that your filters reset on refresh.

**Q (High): What problems does React Query (or SWR) solve that useEffect + useState doesn't?**

Answer: Several critical ones: (1) Request deduplication — two components requesting the same data share one network request; (2) Background refetching on window focus — data stays fresh without manual polling; (3) Cache invalidation — after a mutation, related queries are refetched; (4) Retry on failure with backoff; (5) Race condition prevention — if userId changes while a request is in flight, the stale response is discarded; (6) Shared loading/error states without prop drilling; (7) Optimistic updates with rollback. Implementing all of this correctly with useEffect is possible but tedious and error-prone.

The trap: "I'd just add a loading state and an error state to useState." That handles the happy path but misses deduplication, race conditions, and background sync.

**Q (Medium): Why is form state treated as a separate category from client state? Can't you just use useState?**

Answer: Form state has requirements that generic client state tooling doesn't address: field-level validation, touched/dirty tracking, submission lifecycle (submitting/succeeded/failed), error messages per field, and watch/computed logic between fields. useState works for simple forms but scales poorly — a 20-field form with cross-field validation using useState requires significant manual bookkeeping. Libraries like react-hook-form also solve a performance problem: they avoid re-rendering the entire form on every keystroke by using uncontrolled inputs with a ref-based store under the hood.

The trap: Claiming react-hook-form is only for big forms. The performance benefit of uncontrolled inputs applies even to small forms.

**Q (Medium): How do you avoid overwriting form state with a background server refetch?**

Answer: When populating a form from server data, use `defaultValues` in react-hook-form (or equivalent). `defaultValues` is applied once on mount and is not reactive — background refetches don't affect the form. If you use a reactive `values` prop instead, the form resets whenever the query re-runs, overwriting mid-edit state. If you genuinely want to propagate server changes to an open form (rare), you must explicitly check whether the user has dirtied the field before applying the update.

The trap: Not knowing this distinction exists. Many engineers discover this bug in production when a refetch clears a user's unsaved edits.

**Q (Low): What's the difference between SWR and React Query? When would you pick one over the other?**

Answer: Both solve server state management but with different philosophies. SWR (by Vercel) is smaller, simpler, and opinionated — great for straightforward data fetching with minimal configuration. React Query is more powerful: it handles mutations with optimistic updates and rollback, has richer devtools, supports pagination/infinite scroll natively, and provides finer-grained cache invalidation. For most production applications, React Query's extra capabilities are worth the slightly larger API surface. SWR is a reasonable default for Next.js-centric projects that don't need complex mutation patterns.

The trap: Treating them as identical. The mutation story is meaningfully different.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at this file.

- [ ] Can categorize any piece of state into one of the four types and justify the choice
- [ ] Can explain why server state goes stale and what React Query does about it
- [ ] Can implement URL state with searchParams instead of useState for a filter/pagination scenario
- [ ] Can explain why form state should use `defaultValues` (not reactive `values`) when pre-populating from server data
- [ ] Can name at least three things React Query handles that useEffect + useState misses

---
*Next: Unidirectional vs. Bidirectional Data Flow — understanding the directional flow of state changes is what makes the state categories above predictable to reason about.*
