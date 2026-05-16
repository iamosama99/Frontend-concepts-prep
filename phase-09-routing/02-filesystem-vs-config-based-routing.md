# File-system Based vs. Config-based Routing

## Quick Reference

| | File-system Routing | Config-based Routing |
|---|---|---|
| Route definition | File path = URL path | Explicit route array/object |
| Examples | Next.js, SvelteKit, Nuxt, Remix | React Router, Vue Router, Angular |
| Colocation | Components live next to their route | Components can live anywhere |
| Dynamic routes | `[id].js` / `$id.jsx` syntax | `:id` in route config |
| Flexibility | Convention-driven, less configuration | Full control, more boilerplate |
| Code splitting | Automatic per-file | Manual (lazy()) or automatic (framework-level) |

---

## What Is This?

Routing frameworks define **how URL patterns map to components**. Two dominant philosophies:

**File-system routing** — the directory structure of your `pages/` (or `routes/`, `app/`) folder is the route definition. A file at `pages/products/[id].js` automatically creates the route `/products/:id`. No explicit route config needed.

**Config-based routing** — routes are declared explicitly in a JavaScript array or object, mapping URL patterns to components. You have full control over what maps where, regardless of file location.

```
File-system:                    Config-based:
pages/                          const routes = [
  index.js        → /             { path: '/', component: Home },
  about.js        → /about        { path: '/about', component: About },
  products/                       {
    index.js      → /products       path: '/products',
    [id].js       → /products/:id   children: [
                                      { path: ':id', component: Product }
                                    ]
                                  }
                                ];
```

> **Check yourself:** What is the trade-off between having the file system define your routes versus an explicit config?

---

## Why It Matters

Most modern meta-frameworks (Next.js, SvelteKit, Remix, Nuxt) use file-system routing. If you're joining a team using one of these, understanding the conventions — especially edge cases like catch-all routes, parallel routes, and layout files — is essential. If you're building a SPA without a meta-framework, you'll use config-based routing (React Router, Vue Router) and need to understand its patterns.

---

## File-system Routing

### Convention mapping

```
pages/ (Next.js App Router uses app/)
  index.js            →  /
  about.js            →  /about
  products/
    index.js          →  /products
    [id].js           →  /products/:id         (dynamic segment)
    [...slug].js      →  /products/*           (catch-all)
    [[...slug]].js    →  /products (optional catch-all)
  (marketing)/        →  route group — doesn't affect URL
    landing.js        →  /landing
  @modal/             →  parallel route (Next.js App Router)
    page.js
```

Next.js App Router (file-system by convention):
```
app/
  layout.js           →  root layout (wraps all pages)
  page.js             →  /
  products/
    layout.js         →  layout for /products/* 
    page.js           →  /products
    [id]/
      page.js         →  /products/:id
      loading.js      →  loading UI for this segment
      error.js        →  error boundary for this segment
```

SvelteKit:
```
src/routes/
  +page.svelte        →  /
  +layout.svelte      →  root layout
  about/
    +page.svelte      →  /about
  products/
    [id]/
      +page.svelte    →  /products/:id
      +page.js        →  load function for this page
```

### Dynamic segments

```
// Next.js App Router: app/products/[id]/page.js
export default function ProductPage({ params }) {
  return <div>Product {params.id}</div>;
}

// pages/products/[id].js (Pages Router)
import { useRouter } from 'next/router';
export default function Product() {
  const { query: { id } } = useRouter();
  return <div>Product {id}</div>;
}
```

### Catch-all routes

```
// pages/docs/[...slug].js
// Matches: /docs/intro, /docs/api/overview, /docs/api/v2/reference
export default function Docs({ params }) {
  // params.slug = ['api', 'v2', 'reference']
  return <div>Doc: {params.slug.join('/')}</div>;
}

// pages/[[...slug]].js (optional catch-all)
// Also matches: /  (empty slug = undefined or [])
```

### Route groups (Next.js App Router)

Folders wrapped in `()` create logical groups without affecting the URL:

```
app/
  (auth)/
    login/page.js    →  /login
    signup/page.js   →  /signup
    layout.js        →  layout only for auth pages (no sidebar, etc.)
  (main)/
    dashboard/page.js → /dashboard
    layout.js        →  layout for main app (with sidebar)
```

This lets you share layouts among unrelated URL segments.

---

## Config-based Routing

### React Router v6

```js
import { createBrowserRouter, RouterProvider, Outlet } from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorBoundary />,
    children: [
      { index: true, element: <Home /> },
      { path: 'about', element: <About /> },
      {
        path: 'products',
        element: <ProductsLayout />,
        children: [
          { index: true, element: <ProductList /> },
          {
            path: ':id',
            element: <Product />,
            loader: async ({ params }) => {
              return fetch(`/api/products/${params.id}`).then(r => r.json());
            },
          },
        ],
      },
      { path: '*', element: <NotFound /> },
    ],
  },
]);

function App() {
  return <RouterProvider router={router} />;
}
```

### Vue Router

```js
import { createRouter, createWebHistory } from 'vue-router';

const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/',
      component: () => import('./views/Home.vue'), // lazy-loaded
    },
    {
      path: '/products',
      component: ProductsLayout,
      children: [
        { path: '', component: ProductList },
        { path: ':id', component: Product, props: true },
      ],
    },
    {
      path: '/:pathMatch(.*)*', // catch-all
      component: NotFound,
    },
  ],
});
```

### Named routes and programmatic navigation

```js
// Vue Router: navigate by name instead of path string
router.push({ name: 'product', params: { id: 42 }, query: { tab: 'reviews' } });

// React Router: useNavigate
const navigate = useNavigate();
navigate('/products/42?tab=reviews');
navigate(-1); // back
navigate({ to: '/products', replace: true });
```

---

## Choosing Between Them

**Use file-system routing when:**
- You're in a meta-framework context (Next.js, SvelteKit, Nuxt, Remix) — it's the default and expected
- You want colocation: loading functions, error boundaries, and layouts live next to the routes they affect
- You want automatic per-route code splitting without manual `lazy()` calls

**Use config-based routing when:**
- You're building a SPA without a meta-framework (Vite + React, Vue without Nuxt)
- You need routing logic that doesn't map to a file system cleanly (computed routes, permission-based route trees)
- You're working with a legacy codebase already using React Router or Vue Router

In practice, these aren't in conflict — Next.js uses file-system routing *and* `next/navigation` for programmatic navigation. The distinction is in *how routes are declared*, not how they're used at runtime.

---

## Gotchas

**1. File-system routing is convention-heavy — the conventions differ between frameworks**
`[id]` in Next.js, `[id]` in SvelteKit, `$id` in Remix. The same mental model, different syntax. Read the specific framework's docs before assuming patterns transfer.

**2. Route groups don't create URL segments but do affect layouts**
Using `(auth)` for a layout group means `/login` and `/signup` share a layout, but there's no `/auth/login` path. This is powerful but confusing to newcomers.

**3. Index routes vs. directory routes are different**
In React Router, `{ index: true, element: <Home /> }` renders when the parent path is matched exactly (no child path). An `index.js` in file-system routing serves the same role. Forgetting `index: true` and using `path: ''` instead creates subtly different matching behavior.

**4. Config-based route arrays are evaluated in order (in some routers)**
Vue Router matches the first route that fits. React Router v6 uses ranked matching (more specific wins), but in v5 and many others, order matters. A catch-all `*` route at the top will match everything.

**5. Dynamic segments match anything, including what you don't intend**
`/products/:id` matches `/products/new` — your "create new product" page conflicts with an id named "new". Handle this by placing specific paths before dynamic ones in config routing, or using the framework's route ordering in file-system routing (most prioritize exact matches).

---

## Interview Questions

**Q (High): How does file-system routing work in Next.js App Router? What are route groups and why are they useful?**

Answer: In Next.js App Router, the `app/` directory structure defines routes. A `page.js` file inside a folder creates the route for that path; `layout.js` wraps all routes in that folder and nested folders. `[segment]` folders create dynamic routes; `[...slug]` creates catch-all routes. Route groups — folders wrapped in parentheses like `(auth)` — create organizational groupings without adding a URL segment. This means `/login` and `/signup` can share a layout (e.g., no sidebar, centered card) without being under a `/auth/` prefix in the URL. They're useful for applying different root layouts to different parts of the app without changing URL structure.

**Q (High): What is the difference between a catch-all route and an optional catch-all route?**

Answer: A catch-all route (`[...slug]`) matches one or more path segments. It does not match the parent path with no trailing segments — `/docs/[...slug]` matches `/docs/intro` but not `/docs`. An optional catch-all route (`[[...slug]]`) matches both the parent path alone and any path segments beneath it — `/[[...slug]]` matches `/`, `/about`, and `/about/team`. Optional catch-all is useful for internationalization routes (`/[lang]/...` where the language prefix is optional) or documentation with a variable-depth hierarchy including the index.

**Q (Medium): How do you implement programmatic navigation in React Router v6?**

Answer: Use the `useNavigate` hook inside a component. `const navigate = useNavigate()` returns a function you call with a path string or navigation options. `navigate('/products')` pushes a new entry; `navigate('/products', { replace: true })` replaces the current entry (equivalent to `replaceState`). `navigate(-1)` goes back one step, `navigate(1)` goes forward. For absolute redirects outside of components (in loaders or actions), React Router v6 provides a `redirect()` utility that returns a `Response` with a redirect status code.

**Q (Medium): In config-based routing, what is the difference between `index: true` and `path: ''`?**

Answer: Both match when the parent route's path is matched exactly with no additional segments, but `index: true` is semantically correct and the React Router v6 standard. An index route renders as the default child when no other child route matches. `path: ''` (empty string) also matches the parent path in many routers but can behave differently in edge cases — in React Router v6 specifically, `path: ''` matches differently when nested, whereas `index: true` has precise "render when parent is exactly matched" semantics. Always use `index: true` for the default child in React Router v6.

**Q (Low): Can you use file-system routing and dynamic routing config together?**

Answer: Yes. File-system routing frameworks like Next.js still give you programmatic navigation APIs (`useRouter`, `router.push`). You can also generate routes dynamically at build time via `generateStaticParams` (Next.js) or during SSR, even though the route pattern is defined by the file structure. What you can't do in pure file-system routing is conditionally include or exclude routes based on runtime state — the file structure is fixed at build time. If you need routes that vary based on user permissions at runtime, you'd typically use redirects or access guards rather than altering the route tree itself.

---

## Self-Assessment

- [ ] Explain how dynamic segments and catch-all routes work in file-system routing
- [ ] Describe route groups in Next.js App Router and why you'd use them
- [ ] Implement a basic route config in React Router v6 with nested routes
- [ ] Explain the difference between `index: true` and a path-less route
- [ ] Name two situations where config-based routing is preferable to file-system routing

---
*Next: Nested Routing & Layout Inheritance — how layouts wrap child routes and how to design shared UI structure across route hierarchies.*
