# Data Fetching Strategies: SWR, React Query & Optimistic UI

## Quick Reference

| Concept | Mechanism | Benefit |
|---|---|---|
| stale-while-revalidate | Return cached data immediately, refetch in background | Zero loading spinners on repeat visits |
| Query key | Unique identifier for a cached query | Shared cache across components, targeted invalidation |
| Mutation + invalidation | After mutating, invalidate related query keys | Triggers refetch, keeps UI in sync |
| Optimistic update | Apply expected mutation result before server confirms | Instant perceived response; rollback on error |

## What Is This?

Fetching data in React is deceptively complex. A naive `useEffect` + `fetch` approach handles the happy path but leaves a minefield of edge cases: loading and error states, deduplication when multiple components request the same data, caching to avoid redundant requests, refetching when the window regains focus, retry on failure, background revalidation, and handling race conditions when requests resolve out of order.

SWR (from Vercel) and React Query (TanStack Query) are libraries that handle all of this as a solved problem. They implement a cache-first data fetching model with a `stale-while-revalidate` strategy — return the cached data immediately, then refetch in the background to keep it fresh.

> **Check yourself:** A user opens a tab, loads a page, leaves for 10 minutes, then returns. With React Query's default config, what happens to the query when the window regains focus?

## The Problems with `useEffect` + `fetch`

Before understanding why these libraries exist, you need to feel the pain:

```javascript
// Naive implementation — missing a lot
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(setUser)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [userId]);

  // ...
}
```

This is missing:
- **Race condition:** If `userId` changes while a fetch is in flight, the older request might resolve after the newer one, overwriting fresh data with stale data
- **Deduplication:** If two components mount and both need user 42, they fire two identical requests
- **Caching:** Navigating away and back re-fetches from scratch — always shows a loading state
- **Background revalidation:** No mechanism to re-check data freshness
- **Retry:** No retry on network failure
- **Window focus refetch:** No refetch when user returns to tab

React Query handles all of this, correctly, with a few lines of configuration.

## SWR

SWR (stale-while-revalidate) is Vercel's data fetching hook. The name is its strategy: return stale (cached) data while revalidating (fetching fresh data) in the background.

```javascript
import useSWR from 'swr';

const fetcher = (url) => fetch(url).then(r => r.json());

function UserProfile({ userId }) {
  const { data: user, error, isLoading } = useSWR(`/api/users/${userId}`, fetcher);

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage />;
  return <div>{user.name}</div>;
}
```

**What SWR gives you for free:**
- Cached response returned immediately on repeat visits (no loading spinner)
- Background revalidation on: window focus, network reconnect, and configurable intervals
- Request deduplication (two components with the same key share one request)
- Error retry with exponential backoff
- Mutation support via `mutate()`

**Mutating and revalidating:**
```javascript
const { data: user, mutate } = useSWR('/api/users/42', fetcher);

async function updateUserName(newName) {
  // Optimistic update: immediately show the new name
  mutate({ ...user, name: newName }, false);  // false = don't revalidate yet

  // Call the API
  await fetch('/api/users/42', {
    method: 'PATCH',
    body: JSON.stringify({ name: newName }),
  });

  // Trigger revalidation to sync with server
  mutate();
}
```

SWR is lightweight and opinionated around HTTP semantics. It's the right choice for simpler apps or for teams that want something close to the metal.

## React Query (TanStack Query)

React Query is more feature-rich and better suited for complex data fetching scenarios. It uses a global cache keyed by query keys and has more sophisticated cache management.

```javascript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

function UserProfile({ userId }) {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],    // cache key — array, not string
    queryFn: () => fetch(`/api/users/${userId}`).then(r => r.json()),
    staleTime: 5 * 60 * 1000,     // consider data fresh for 5 minutes
    gcTime: 10 * 60 * 1000,       // keep in cache for 10 minutes after no subscribers
  });

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage error={error} />;
  return <div>{user.name}</div>;
}
```

### Query Keys

Query keys are the fundamental unit of cache organization. They're arrays that uniquely identify a piece of server state:

```javascript
// Different keys → different cache entries
['user', 42]           // user 42
['user', 43]           // user 43 — separate entry
['users', { page: 1 }] // users page 1
['users', { page: 2 }] // users page 2

// Invalidating by prefix: invalidate everything about users
queryClient.invalidateQueries({ queryKey: ['users'] });
// This invalidates ['users'], ['users', {page:1}], ['users', {page:2}], etc.
```

### Mutations and Cache Invalidation

After a mutation succeeds, you need the related queries to refetch. React Query's mutation system handles this:

```javascript
const queryClient = useQueryClient();

const updateUser = useMutation({
  mutationFn: ({ id, data }) =>
    fetch(`/api/users/${id}`, { method: 'PATCH', body: JSON.stringify(data) })
      .then(r => r.json()),

  onSuccess: (updatedUser) => {
    // Update the cache directly with the returned data — no extra fetch
    queryClient.setQueryData(['user', updatedUser.id], updatedUser);

    // Also invalidate any lists that contain this user
    queryClient.invalidateQueries({ queryKey: ['users'] });
  },
});
```

### `staleTime` vs. `gcTime` (formerly `cacheTime`)

This distinction trips up most React Query users:

- **`staleTime`:** How long the data is considered fresh. During this window, the query won't refetch in the background (window focus won't trigger revalidation, etc.). Default: 0 (data is immediately stale — always refetch in background).
- **`gcTime`:** How long the data stays in the cache after there are no active subscribers. After this, the cache entry is garbage collected. Default: 5 minutes.

```javascript
useQuery({
  queryKey: ['products'],
  queryFn: fetchProducts,
  staleTime: 5 * 60 * 1000,   // Don't refetch if data is less than 5 minutes old
  gcTime: 10 * 60 * 1000,     // Keep in cache for 10 minutes after no observers
});
```

Practical guidance: set `staleTime` based on how quickly the data changes and how much you care about staleness. For slowly changing reference data (countries list, categories), set a high `staleTime`. For frequently changing data (user notifications), keep `staleTime: 0`.

## Optimistic UI

Optimistic UI means applying the expected result of a mutation to the UI immediately — before the server confirms — then reconciling with the actual server response.

The user experience: clicking "Like" immediately shows the liked state and incremented count. If the server confirms, nothing changes (the optimistic state was correct). If the server fails, the UI rolls back to the previous state.

### The Pattern

```javascript
const queryClient = useQueryClient();

const likeMutation = useMutation({
  mutationFn: (postId) => fetch(`/api/posts/${postId}/like`, { method: 'POST' }),

  onMutate: async (postId) => {
    // 1. Cancel any in-flight refetches for this query (avoid race conditions)
    await queryClient.cancelQueries({ queryKey: ['post', postId] });

    // 2. Snapshot the current value for rollback
    const previousPost = queryClient.getQueryData(['post', postId]);

    // 3. Optimistically update the cache
    queryClient.setQueryData(['post', postId], (old) => ({
      ...old,
      liked: true,
      likeCount: old.likeCount + 1,
    }));

    // 4. Return context with snapshot for rollback
    return { previousPost };
  },

  onError: (err, postId, context) => {
    // 5. On error, roll back to snapshot
    queryClient.setQueryData(['post', postId], context.previousPost);
  },

  onSettled: (data, err, postId) => {
    // 6. Always refetch to sync with true server state
    queryClient.invalidateQueries({ queryKey: ['post', postId] });
  },
});

// Usage — instant UI response
likeButton.onClick = () => likeMutation.mutate(postId);
```

### When Optimistic UI Is Appropriate

Optimistic updates work best when:
- The operation is very likely to succeed (>99% success rate)
- The expected result is deterministic (you know what the server will return)
- The visual feedback matters for perceived responsiveness (social actions, toggles, reorders)

Optimistic UI is wrong when:
- The operation has a significant failure rate (payment processing, availability-dependent booking)
- The expected result depends on server computation the client can't predict (pricing with server-side discounts, validation-heavy forms)
- Rolling back would be confusing to users (don't optimistically show a sent email if it might fail)

### Rollback Complexity

The harder part of optimistic UI is designing the rollback experience:

```javascript
onError: (err, variables, context) => {
  // Restore snapshot
  queryClient.setQueryData(queryKey, context.previousData);

  // Show an error toast explaining the rollback
  toast.error('Failed to save. Your changes have been undone.');
  // Don't silently revert — the user's action disappeared. They need to know.
}
```

The rollback must be visible. Silently reverting optimistic changes causes confusion — the user sees their action "undo itself" with no explanation.

## Prefetching

React Query allows prefetching data before it's needed — during hover on a link, for example:

```javascript
// Prefetch on hover — data is in cache when user navigates
function ProductLink({ productId, children }) {
  const queryClient = useQueryClient();

  const prefetch = () => {
    queryClient.prefetchQuery({
      queryKey: ['product', productId],
      queryFn: () => fetchProduct(productId),
      staleTime: 60_000,  // don't prefetch if cache is fresh
    });
  };

  return (
    <Link to={`/products/${productId}`} onMouseEnter={prefetch}>
      {children}
    </Link>
  );
}
```

When the user actually navigates to the product page, the data is already in cache — the component renders immediately with no loading state.

## Gotchas

**`staleTime: 0` (the default) causes a background fetch on every mount.** Every component mount triggers a background refetch if the data is stale (which it always is at `staleTime: 0`). This is intentional — always show fresh data — but can cause unnecessary requests if the data genuinely doesn't change often. Set `staleTime` deliberately.

**Query key stability matters.** If a query key contains an object or function that's recreated on every render (`queryKey: ['users', { filter }]` where `filter` is a new object each render), React Query treats it as a new query every render. Use stable references or `useMemo` for object query keys.

**Optimistic updates break when multiple concurrent mutations happen.** If a user clicks "Like" twice quickly, two concurrent optimistic updates fight over the same cache entry. Handle this by using a debounce, disabling the button during mutation, or using atomic operations at the server level.

**Error boundaries and React Query:** React Query doesn't integrate with React error boundaries by default. Use `throwOnError: true` in your query options to throw errors that can be caught by error boundaries, rather than handling all errors inline.

**`invalidateQueries` triggers a background refetch, not an immediate one.** After calling `invalidateQueries`, the query is marked stale — it refetches in the background (showing stale data until fresh data arrives) if there are active subscribers. If you need the fresh data synchronously, use `refetchQueries` instead.

## Interview Questions

**Q (High): What is stale-while-revalidate and how does React Query implement it?**

Answer: Stale-while-revalidate is an HTTP caching strategy (RFC 5861) where a cached response is returned immediately even if it's stale, while a fresh response is fetched in the background. React Query implements this at the JavaScript level: when a component mounts and requests data that's in the cache but stale (older than `staleTime`), React Query returns the cached data immediately — no loading state, no blank screen — while simultaneously triggering a background fetch. When the fresh data arrives, the component updates. The result: instant UI on repeat visits, always trending toward fresh data. React Query also triggers background revalidation on window focus (user returns to the tab), network reconnection, and configurable polling intervals. This is why React Query apps feel faster than `useEffect` + `fetch` apps — loading states only appear on the very first request, not on every navigation.

The trap: Describing SWR as "just a hook that fetches data." The caching model and when background revalidation triggers are the actual substance.

---

**Q (High): Explain the full optimistic update pattern in React Query, including rollback.**

Answer: Optimistic UI in React Query uses the `onMutate`, `onError`, and `onSettled` callbacks. In `onMutate`: (1) cancel any in-flight refetches for the affected queries to prevent a race condition where a refetch overwrites the optimistic update; (2) snapshot the current cache value for rollback; (3) write the expected mutation result to the cache directly via `setQueryData`; (4) return the snapshot as context. If the mutation succeeds, the optimistic update remains (or is replaced by the server response). If the mutation fails, `onError` receives the snapshot from context and restores it with `setQueryData`. In `onSettled` (runs on both success and error), call `invalidateQueries` to trigger a background fetch of true server state. The rollback must be accompanied by visible user feedback — a toast error — because silently reverting optimistic changes is disorienting.

The trap: Describing optimistic updates without mentioning the `cancelQueries` step. This is the race condition prevention — without it, a concurrent background refetch can overwrite the optimistic state.

---

**Q (High): What is the difference between `staleTime` and `gcTime` in React Query?**

Answer: `staleTime` controls freshness: how long after a fetch the data is considered fresh. During this window, the query won't trigger background revalidation (no refetch on window focus, no automatic background fetch). After `staleTime`, the data is "stale" — not invalid or removed, just eligible for background revalidation. Default is 0, meaning data is immediately stale and always eligible for background revalidation. `gcTime` (formerly `cacheTime`) controls persistence: how long after there are no active subscribers the data stays in the cache before being garbage collected. A query with no active `useQuery` subscribers still lives in cache for `gcTime` duration — so navigating away and back quickly shows cached data. Default is 5 minutes. Key distinction: stale data is still served from cache (immediately, while refetching); garbage-collected data is gone and the next request shows a loading state again.

The trap: Confusing "stale" with "removed." Stale data is still in cache and still returned — stale just means eligible for background revalidation.

---

**Q (Medium): What is request deduplication and why does it matter?**

Answer: Request deduplication means that if multiple components simultaneously request the same data (same query key), only one network request is made — all components share the result. Without deduplication, a page that renders 3 components all calling `useQuery(['user', 42])` fires 3 parallel identical requests, wasting bandwidth and creating race conditions if they return out of order. React Query deduplicates by query key: when a query is in-flight, any other component that needs the same key is "subscribed" to that in-flight request rather than starting a new one. All subscribers receive the same response simultaneously. This is a significant performance improvement over `useEffect` + `fetch` on complex pages where many components need the same data.

The trap: Not knowing deduplication is happening. Candidates who've only used `useEffect` often don't realize redundant requests are a solvable problem, not just a fact of life.

---

**Q (Low): How does React Query handle background refetching when the window regains focus, and how would you disable it?**

Answer: React Query subscribes to `document.addEventListener('visibilitychange')` and `window.addEventListener('focus')`. When the window becomes visible or focused after being hidden/blurred, React Query marks all active queries as stale (regardless of `staleTime`) and triggers background refetches. This ensures users always see fresh data when they return to a tab. Disable globally in the `QueryClient` config: `defaultOptions: { queries: { refetchOnWindowFocus: false } }`. Disable per-query: `useQuery({ ..., refetchOnWindowFocus: false })`. When to disable: for data you know doesn't change (static reference data), to reduce server load during development, or for queries where revalidation is disruptive (e.g., a form that would reset). The default behavior is correct for most applications.

The trap: Not knowing this behavior exists and being surprised by "extra" network requests. Understanding what React Query does automatically vs. what's configurable is a senior-level expectation.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain stale-while-revalidate: return cached data immediately, refetch in background
- [ ] Can explain the difference between `staleTime` and `gcTime`
- [ ] Can describe the full optimistic update pattern: onMutate (cancel, snapshot, apply), onError (rollback), onSettled (invalidate)
- [ ] Can explain why `cancelQueries` is necessary before an optimistic update
- [ ] Can explain what query keys are and how prefix-based invalidation works
- [ ] Can describe request deduplication and why it matters on a component-heavy page

---
*Next: Caching Layers — HTTP Cache, Service Worker Cache, and CDN — the layered caching model that sits between your server and your users.*

*See also: [Phase 4 — Server, Client, URL & Form State](../phase-04-state-management/01-server-client-url-form-state.md) — establishes the taxonomy that explains *why* React Query exists: server state has fundamentally different characteristics (staleness, async, shared ownership) from client state, and needs a dedicated tool to manage them.*
