# Navigation Guards & Middleware

## Quick Reference

| Framework | Mechanism | Where it runs |
|---|---|---|
| React Router v6 | `loader` functions, layout redirect | Client (and server in Remix) |
| Vue Router | `beforeEach`, `beforeEnter`, component hooks | Client |
| Next.js | `middleware.js` | Edge runtime (server, before page) |
| Angular | `CanActivate`, `CanDeactivate` guards | Client |
| SvelteKit | `+page.server.js` load + redirect | Server |

---

## What Is This?

Navigation guards intercept route transitions and decide whether to allow, redirect, or cancel them — before the destination component mounts. Common use cases: authentication checks, permission verification, unsaved-changes warnings, and analytics tracking.

```
User navigates to /dashboard
         │
         ▼
    [Guard runs]
         │
    ┌────┴────┐
    │         │
  Allow     Redirect
    │         │
    ▼         ▼
/dashboard  /login
```

> **Check yourself:** Why is it safer to implement auth guards at the route level rather than inside each page component?

---

## Why It Matters

Authentication and authorization at the route level is a core pattern in production apps. Understanding the differences between client-side guards, server-side middleware, and "redirect in loader" patterns — and the security implications of each — is expected knowledge for senior frontend roles.

---

## React Router v6 — Loaders and Layout Redirect

React Router doesn't have a traditional "guard" API. Instead, protection is implemented via route loaders and pathless layout routes.

### Loader-based protection

```js
import { redirect } from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/dashboard',
    loader: async () => {
      const user = await getAuthenticatedUser();
      if (!user) {
        throw redirect('/login');  // stops render, sends user to /login
      }
      return { user };
    },
    element: <Dashboard />,
  },
]);
```

The loader runs before the component renders. `throw redirect('/login')` is a Response — React Router catches it and performs the redirect. The component never mounts if the loader throws.

### Pathless layout route guard

```jsx
function RequireAuth() {
  const { user, loading } = useAuth();

  if (loading) return <PageSpinner />;
  if (!user) return <Navigate to="/login" replace state={{ from: location }} />;

  return <Outlet />;
}

const router = createBrowserRouter([
  {
    element: <RequireAuth />,  // no path — wraps children
    children: [
      { path: '/dashboard', element: <Dashboard /> },
      { path: '/settings', element: <Settings /> },
    ],
  },
  { path: '/login', element: <Login /> },
]);
```

`<Navigate replace>` performs the redirect. `state={{ from: location }}` preserves the original destination so after login you can redirect back:

```jsx
function Login() {
  const location = useLocation();
  const navigate = useNavigate();
  const from = location.state?.from?.pathname || '/dashboard';

  async function handleLogin(credentials) {
    await login(credentials);
    navigate(from, { replace: true });
  }
  // ...
}
```

### Role-based protection

```jsx
function RequireRole({ role }) {
  const { user } = useAuth();
  if (!user) return <Navigate to="/login" replace />;
  if (!user.roles.includes(role)) return <Navigate to="/unauthorized" replace />;
  return <Outlet />;
}

const router = createBrowserRouter([
  {
    element: <RequireRole role="admin" />,
    children: [
      { path: '/admin', element: <AdminPanel /> },
    ],
  },
]);
```

---

## Vue Router — Navigation Guards

Vue Router has a rich guard API at multiple levels.

### Global guards

```js
const router = createRouter({ routes });

// beforeEach: runs before every navigation
router.beforeEach(async (to, from) => {
  const authStore = useAuthStore();

  // Allow navigating to public routes without auth
  if (to.meta.public) return true;

  if (!authStore.isAuthenticated) {
    // Redirect to login, passing intended destination
    return { name: 'login', query: { redirect: to.fullPath } };
  }

  // Role check
  if (to.meta.role && !authStore.user.roles.includes(to.meta.role)) {
    return { name: 'unauthorized' };
  }

  return true; // allow navigation
});

// afterEach: runs after navigation completes (no ability to redirect)
router.afterEach((to, from) => {
  document.title = to.meta.title || 'My App';
  analytics.track('page_view', { path: to.path });
});
```

Route meta fields carry guard metadata:

```js
const routes = [
  { path: '/login', component: Login, meta: { public: true } },
  {
    path: '/admin',
    component: AdminPanel,
    meta: { role: 'admin', title: 'Admin Panel' },
  },
];
```

### Per-route guards

```js
const routes = [
  {
    path: '/admin',
    component: AdminPanel,
    beforeEnter: (to, from) => {
      const { user } = useAuthStore();
      if (!user?.roles.includes('admin')) {
        return { name: 'unauthorized' };
      }
    },
  },
];
```

### Component-level guards

```js
// Options API
export default {
  beforeRouteEnter(to, from, next) {
    // Called before the component is created — no access to `this`
    next(vm => {
      // `vm` is the component instance, available in callback
      vm.fetchData();
    });
  },

  beforeRouteUpdate(to, from) {
    // Called when route changes but component is reused (e.g., /users/1 → /users/2)
    this.fetchUser(to.params.id);
  },

  beforeRouteLeave(to, from) {
    // Called before leaving this route
    if (this.hasUnsavedChanges) {
      const confirmed = window.confirm('You have unsaved changes. Leave anyway?');
      if (!confirmed) return false; // cancel navigation
    }
  },
};
```

```js
// Composition API
import { onBeforeRouteLeave, onBeforeRouteUpdate } from 'vue-router';

export default {
  setup() {
    const isDirty = ref(false);

    onBeforeRouteLeave(() => {
      if (isDirty.value) {
        return window.confirm('Unsaved changes — leave?');
      }
    });

    onBeforeRouteUpdate(async (to) => {
      await fetchUser(to.params.id);
    });
  },
};
```

### Guard resolution order

For a navigation to `/admin`:

1. `router.beforeEach` (global)
2. `beforeEnter` on the route (per-route)
3. `beforeRouteEnter` in the component (component-level)
4. `router.afterEach` (global, after navigation)

---

## Next.js Middleware

Next.js `middleware.js` runs on the Edge runtime (V8 isolate, not Node.js) before every request. It can rewrite, redirect, or modify the request/response. It runs before SSR, before static file serving, before route handlers.

```js
// middleware.js (at project root)
import { NextResponse } from 'next/server';

export function middleware(request) {
  const token = request.cookies.get('auth-token')?.value;
  const isPublicPath = ['/login', '/signup', '/api/auth'].some(path =>
    request.nextUrl.pathname.startsWith(path)
  );

  if (!isPublicPath && !token) {
    const loginUrl = new URL('/login', request.url);
    loginUrl.searchParams.set('redirect', request.nextUrl.pathname);
    return NextResponse.redirect(loginUrl);
  }

  return NextResponse.next(); // allow request through
}

// Only run middleware for these paths (skip static assets, _next, etc.)
export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
};
```

Middleware operates on the request before any server component or page loads. For full token validation (checking JWT signature, hitting a database), use server components or API routes instead — middleware should be fast (no DB calls).

---

## Unsaved Changes Guard

A common pattern: warn users who try to navigate away from a form with unsaved changes.

```jsx
// React — custom hook using beforeunload and router navigation
function useUnsavedChangesWarning(isDirty) {
  const navigate = useNavigate();

  useEffect(() => {
    if (!isDirty) return;

    // Browser's built-in prompt for tab close / hard navigation
    const handleBeforeUnload = (e) => {
      e.preventDefault();
      e.returnValue = '';
    };
    window.addEventListener('beforeunload', handleBeforeUnload);

    return () => window.removeEventListener('beforeunload', handleBeforeUnload);
  }, [isDirty]);

  // For SPA navigation — wrap navigate to confirm first
  const safeNavigate = useCallback((to, options) => {
    if (isDirty) {
      const confirmed = window.confirm('You have unsaved changes. Leave?');
      if (!confirmed) return;
    }
    navigate(to, options);
  }, [isDirty, navigate]);

  return safeNavigate;
}
```

React Router v6.3+ adds an experimental `unstable_useBlocker` / `useBlocker` API:

```jsx
import { useBlocker } from 'react-router-dom';

function EditForm() {
  const isDirty = useFormDirty();

  useBlocker(() => {
    if (!isDirty) return false; // allow navigation
    return !window.confirm('Leave without saving?'); // block if not confirmed
  });

  // ...
}
```

---

## Security Consideration

Client-side guards are UX safeguards, not security mechanisms. A user can open DevTools and call `navigate('/admin')` to bypass any client-side guard. Real authorization must be enforced on the server:
- API endpoints return 401/403 for unauthorized requests
- Server components check authentication before rendering sensitive data
- Next.js middleware can redirect at the edge, but token validation should happen server-side

Use client-side guards to prevent navigation mistakes and improve UX; use server-side auth to prevent unauthorized data access.

---

## Gotchas

**1. Vue Router's `beforeEach` runs for the initial navigation**
Don't assume the user "just navigated here" — the guard also fires when the app first loads and initializes the route. Make sure the auth state is available at that point (initialize Pinia/Vuex before creating the router).

**2. Returning `false` from a Vue guard cancels navigation but doesn't revert browser URL**
If the user used the browser's back button and the guard returns `false`, the URL bar has already changed back. The browser doesn't automatically revert it. Handle this edge case explicitly or use `router.replace` to correct the URL.

**3. `loader`-based redirects in React Router throw Responses, not return them**
`throw redirect('/login')` is the correct pattern — `return redirect('/login')` also works but the behavior differs in error boundaries. Throwing makes the redirect unambiguous and consistent with error handling.

**4. Next.js middleware runs on the Edge — no Node.js APIs**
No `fs`, no `Buffer` (limited support), no full `crypto`. You can verify JWTs using the Web Crypto API or lightweight edge-compatible libraries, but you cannot make database queries or use heavy Node.js modules.

**5. Component-level `beforeRouteEnter` has no access to `this`**
In Vue's Options API, `beforeRouteEnter` is called before the component instance exists. You must use the `next(vm => ...)` callback to access the component after it's created. `beforeRouteUpdate` and `beforeRouteLeave` do have access to `this`.

---

## Interview Questions

**Q (High): How do you implement authentication protection in React Router v6?**

Answer: Two approaches: loader-based and layout-route-based. In the loader approach, add an async `loader` function to the route that checks auth state and `throw redirect('/login')` if the user isn't authenticated. This runs before the component mounts, preventing any flash of protected content. In the layout-route approach, create a pathless route with `element: <RequireAuth />` that checks auth in render and returns `<Navigate to="/login" replace>` if not authenticated, or `<Outlet>` if authenticated — all protected routes become children of this wrapper. The loader approach integrates better with React Router's data fetching (no render-then-redirect flash), while the layout approach is simpler for client-side-only auth (e.g., reading from a React context).

**Q (High): Explain the Vue Router guard resolution order for a full navigation.**

Answer: When navigating to a new route: (1) component-level `beforeRouteLeave` fires on the current component; (2) global `beforeEach` guards fire; (3) per-route `beforeEnter` fires on the destination route; (4) component-level `beforeRouteEnter` fires for the incoming component; (5) navigation is confirmed; (6) `beforeRouteUpdate` fires if any component is being reused with new params; (7) `afterEach` global hooks fire. If any guard in steps 1–4 returns `false` or a redirect destination, the navigation is cancelled (or redirected) and subsequent guards don't run.

**Q (High): What is Next.js middleware and how does it differ from a page-level auth check?**

Answer: Next.js middleware (`middleware.js`) is a function that runs on the Edge runtime before every request — before routing, before SSR, before any page code executes. It can redirect, rewrite, or modify the request/response. A page-level auth check (inside a server component or `getServerSideProps`) runs after the request has been routed to the correct page and that page's code has begun executing. Middleware is faster and more centralized — one place to protect all routes — but it's limited to the Edge runtime (no Node.js APIs, no DB calls). Page-level checks have full server capabilities but are per-page and run later in the request lifecycle. Use middleware for fast token presence checks (cookie/header exists); use server-side page code for full validation (JWT signature verification, database session check).

**Q (Medium): How do you implement an unsaved-changes warning when the user navigates away from a form?**

Answer: Two boundaries to handle: soft navigation (SPA route change) and hard navigation (tab close, browser back to different origin). For hard navigation, add a `beforeunload` event listener that sets `event.returnValue = ''` — this triggers the browser's native "Leave page?" dialog. For SPA navigation, either use React Router's `useBlocker` hook (v6.7+) which intercepts programmatic navigation and link clicks, or wrap your `navigate` calls in a function that checks the dirty state first and shows a `window.confirm` before proceeding. The `beforeunload` event fires for hard navigations only; `useBlocker` fires for SPA navigation only. You need both for complete coverage.

**Q (Low): Why are client-side navigation guards not a security mechanism?**

Answer: Client-side guards run in the browser, where the user has full control. A user can open DevTools and call the routing API directly, modify JavaScript in memory, or make HTTP requests to protected API endpoints directly — bypassing the router entirely. Guards prevent accidental navigation (a user clicking a link they shouldn't) but can't stop a determined attacker. Security must be enforced server-side: API endpoints must verify tokens and return 401/403 for unauthorized requests; server-rendered pages must check auth before returning sensitive HTML; server-side middleware (Next.js middleware, server guards) can redirect at the network level. Client-side guards are for UX, not security.

---

## Self-Assessment

- [ ] Implement a React Router v6 auth guard using both the loader pattern and the layout route pattern
- [ ] Write a Vue Router `beforeEach` global guard that checks auth and role
- [ ] Explain the Vue Router guard resolution order from route leave to after navigation
- [ ] Describe what Next.js middleware does and what it cannot do (Edge runtime limits)
- [ ] Implement an unsaved-changes warning that covers both SPA and hard navigation

---
*Next: URL as State — deep linking, shareability, and encoding application state in the URL.*
