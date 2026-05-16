# Nested Routing & Layout Inheritance

## Quick Reference

| Concept | Purpose | Key mechanism |
|---|---|---|
| Nested routes | Model URL hierarchy as a component tree | Parent renders `<Outlet>` / `<slot>` |
| Shared layout | Wrap child routes in persistent UI (nav, sidebar) | Layout component renders at parent route level |
| Index route | Default content when parent is matched exactly | `index: true` in config, `index.js` in FS routing |
| Layout files | File-system layout wrapping (`layout.js`) | Automatically wraps all pages in directory |
| Parallel routes | Render multiple independent route segments simultaneously | `@slot` folders (Next.js App Router) |

---

## What Is This?

Nested routing models the hierarchical relationship between URLs and UI. When a user visits `/dashboard/settings`, they expect the outer dashboard shell (navigation bar, sidebar) and the settings content simultaneously. Nested routing makes this explicit: the `dashboard` route owns the shell layout, and `settings` is a child route that renders inside it.

The parent route renders an `<Outlet>` (React Router) or `<RouterView>` (Vue Router) — a placeholder that the active child route fills at runtime.

```
URL: /dashboard/settings

Component tree:
  <App>              (router root)
    <DashboardLayout>  (matches /dashboard)
      ← nav, sidebar here
      <SettingsPage>   (matches /dashboard/settings, rendered in Outlet)
```

> **Check yourself:** What happens if a parent route doesn't render `<Outlet>`?

---

## Why It Matters

Nested routing is the core mechanism behind layout inheritance in every major frontend framework. Understanding how `Outlet`, layout files, and index routes interact is prerequisite knowledge for designing maintainable route hierarchies and for answering questions about how frameworks like Next.js App Router, Remix, and React Router actually work.

---

## React Router v6 — Nested Routes

### Basic nesting

```jsx
import { createBrowserRouter, RouterProvider, Outlet, Link } from 'react-router-dom';

function DashboardLayout() {
  return (
    <div>
      <nav>
        <Link to="/dashboard/overview">Overview</Link>
        <Link to="/dashboard/settings">Settings</Link>
      </nav>
      <main>
        <Outlet /> {/* child route renders here */}
      </main>
    </div>
  );
}

const router = createBrowserRouter([
  {
    path: '/dashboard',
    element: <DashboardLayout />,
    children: [
      { index: true, element: <DashboardOverview /> }, // /dashboard exactly
      { path: 'settings', element: <SettingsPage /> }, // /dashboard/settings
      { path: 'users/:id', element: <UserDetail /> },  // /dashboard/users/42
    ],
  },
]);
```

### Deeply nested

Routes can nest to any depth. Each layout renders an `<Outlet>` for its children:

```jsx
const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,         // renders <Header> + <Outlet>
    children: [
      {
        path: 'dashboard',
        element: <DashboardLayout />, // renders <Sidebar> + <Outlet>
        children: [
          { index: true, element: <Overview /> },
          {
            path: 'reports',
            element: <ReportsLayout />, // renders <ReportNav> + <Outlet>
            children: [
              { index: true, element: <ReportList /> },
              { path: ':reportId', element: <ReportDetail /> },
            ],
          },
        ],
      },
    ],
  },
]);
```

URL `/dashboard/reports/42` renders: `RootLayout → DashboardLayout → ReportsLayout → ReportDetail`

### Layout routes (pathless)

A route with no `path` acts as a layout wrapper without consuming a URL segment:

```jsx
const router = createBrowserRouter([
  {
    // No path — just wraps children with authentication check
    element: <RequireAuth />,
    children: [
      { path: '/dashboard', element: <Dashboard /> },
      { path: '/profile', element: <Profile /> },
    ],
  },
  { path: '/login', element: <Login /> },
]);

function RequireAuth() {
  const { user } = useAuth();
  if (!user) return <Navigate to="/login" replace />;
  return <Outlet />;
}
```

This separates the authentication concern from URL structure — `/dashboard` and `/profile` are top-level URLs, but they share an auth wrapper.

---

## Next.js App Router — Layout Files

In the App Router, `layout.js` files automatically wrap all `page.js` files in the same directory and all subdirectories.

```
app/
  layout.js          ← root layout (wraps everything)
  page.js            ← renders at /
  dashboard/
    layout.js        ← wraps /dashboard and all /dashboard/* pages
    page.js          ← renders at /dashboard
    settings/
      page.js        ← renders at /dashboard/settings
```

Layout files receive a `children` prop (equivalent to `<Outlet>`):

```jsx
// app/dashboard/layout.js
export default function DashboardLayout({ children }) {
  return (
    <div className="dashboard">
      <Sidebar />
      <main>{children}</main>
    </div>
  );
}
```

Layouts are persistent — navigating between `/dashboard/settings` and `/dashboard/users` does not remount `DashboardLayout`. Only the page content changes. This is the App Router's mechanism for maintaining scroll position, preserving form state, and avoiding layout flicker on navigation.

---

## Vue Router — Nested Routes

```js
const router = createRouter({
  routes: [
    {
      path: '/dashboard',
      component: DashboardLayout, // must render <RouterView>
      children: [
        { path: '', component: DashboardOverview },  // /dashboard
        { path: 'settings', component: Settings },   // /dashboard/settings
        {
          path: 'reports',
          component: ReportsLayout,
          children: [
            { path: '', component: ReportList },
            { path: ':id', component: ReportDetail, props: true },
          ],
        },
      ],
    },
  ],
});
```

```vue
<!-- DashboardLayout.vue -->
<template>
  <div class="dashboard">
    <Sidebar />
    <RouterView /> <!-- child route renders here -->
  </div>
</template>
```

### Named views

Vue Router supports rendering multiple views at the same level (not nested):

```js
{
  path: '/dashboard',
  components: {
    default: Dashboard,
    sidebar: DashboardSidebar,  // renders in <RouterView name="sidebar">
    header: DashboardHeader,    // renders in <RouterView name="header">
  },
}
```

```vue
<RouterView name="header" />
<RouterView />          <!-- default view -->
<RouterView name="sidebar" />
```

---

## Parallel Routes (Next.js App Router)

App Router supports rendering multiple independent route segments at the same URL — the equivalent of named views. Define a slot with `@slot-name`:

```
app/
  layout.js
  page.js
  @modal/
    page.js      ← rendered in the @modal slot
  @sidebar/
    page.js      ← rendered in the @sidebar slot
```

```jsx
// app/layout.js
export default function Layout({ children, modal, sidebar }) {
  return (
    <div>
      <aside>{sidebar}</aside>
      <main>{children}</main>
      {modal}
    </div>
  );
}
```

Parallel routes are used for modals that have their own URL (navigating to `/photos/42` opens a modal without losing the underlying page context) and for dashboards with independently navigable sections.

---

## Index Routes

An index route renders when the parent path is matched exactly and no child path is active.

```jsx
// React Router v6
{
  path: '/dashboard',
  element: <DashboardLayout />,
  children: [
    { index: true, element: <Overview /> },  // /dashboard exactly
    { path: 'settings', element: <Settings /> },
  ],
}
```

Without an index route, visiting `/dashboard` exactly renders `DashboardLayout` with an empty `<Outlet>` — nothing fills the main content area.

In file-system routing, `index.js` or `index.tsx` serves the same purpose: it's the page rendered when the directory's path is matched exactly.

---

## Outlet Context

React Router's `<Outlet>` can pass context to child routes, avoiding prop drilling through route boundaries:

```jsx
function DashboardLayout() {
  const { user } = useAuth();
  return (
    <div>
      <Sidebar user={user} />
      <Outlet context={{ user }} /> {/* pass data to child routes */}
    </div>
  );
}

// In a child route:
function Settings() {
  const { user } = useOutletContext();
  return <div>Settings for {user.name}</div>;
}
```

---

## Gotchas

**1. Forgetting `<Outlet>` in a parent layout silently drops child routes**
If `DashboardLayout` doesn't render `<Outlet>`, navigating to `/dashboard/settings` mounts `DashboardLayout` but `SettingsPage` never appears. There's no error — the child component just doesn't render. This is the most common mistake when setting up nested routes.

**2. Pathless layout routes apply to all children — including ones you didn't intend**
A layout route with `element: <RequireAuth />` and no `path` wraps all its children. If you accidentally nest a public route inside it, it gets protected. Be explicit about which routes belong inside auth wrappers.

**3. Next.js layouts are persistent — they don't re-run server components on same-segment navigation**
In App Router, navigating from `/dashboard/settings` to `/dashboard/profile` does not re-run `app/dashboard/layout.js`. If you fetch user data in the layout server component, it won't be refreshed by navigation between children. This is intentional (performance), but means layouts aren't suitable for data that changes per child route.

**4. Vue Router's `<RouterView>` in wrong location renders at wrong depth**
If you place `<RouterView>` in a grandchild template instead of the direct parent, the router doesn't know to render children there. The component tree must exactly mirror the route config tree.

**5. Index routes and empty-path children have subtly different behavior**
`{ path: '', component: Home }` and `{ index: true, component: Home }` both match the parent path, but `path: ''` matches as a "redirect to self" in some router versions and may differ on trailing slash handling. Always use `index: true` in React Router v6.

---

## Interview Questions

**Q (High): How does `<Outlet>` work in React Router v6 and what happens if you omit it?**

Answer: `<Outlet>` is a placeholder that React Router fills with the currently active child route's element. When the router matches a URL, it builds a component tree from the matched route hierarchy. Each parent route's element must render `<Outlet>` for the matched child to appear. If you omit it, the parent layout mounts as expected but the child route's component is silently discarded — no error is thrown, nothing renders in place of the missing Outlet. The fix is straightforward: add `<Outlet />` wherever in the parent's JSX the child content should appear.

**Q (High): How does layout persistence work in Next.js App Router and why does it matter?**

Answer: In App Router, `layout.js` files are not remounted when navigating between child pages. If you navigate from `/dashboard/settings` to `/dashboard/profile`, the `DashboardLayout` component remains mounted — React doesn't tear it down and recreate it. Only the `children` prop (the active page) changes. This has important implications: state inside the layout (open/closed sidebar, scroll position, form state) persists across navigation. It also means server component layouts don't re-fetch data on child navigation. This is why App Router layouts are much better for persistent UI shells (navigation, sidebars, authenticated user data) than page-level components that re-mount on every navigation.

**Q (High): What is a pathless layout route and what is it used for?**

Answer: A pathless layout route is a route in React Router v6 with no `path` property — it has only `element` and `children`. It wraps its children in a layout component without consuming any URL segment. Its children are still accessible at their own paths. The primary use case is applying shared logic or UI to a group of routes without grouping them under a common URL prefix. Authentication guards are the canonical example: `element: <RequireAuth />` wraps protected routes; users who aren't authenticated get redirected to `/login`, while authenticated users see the child route via `<Outlet>`. The URL stays clean (`/dashboard`, not `/auth/dashboard`), but the protection is enforced at the layout level.

**Q (Medium): How do you pass data from a parent layout to child routes in React Router without prop drilling?**

Answer: Use `<Outlet context={value} />` in the parent layout to pass any value (object, function, etc.) down to the active child route. In the child, call `useOutletContext()` to read it. This avoids prop drilling through route boundaries and is more composable than a global store for route-scoped data like the authenticated user, a currently selected workspace, or a shared data-fetching result. The context is typed in TypeScript if you wrap `useOutletContext` with a generic: `useOutletContext<{ user: User }>()`.

**Q (Low): What are parallel routes in Next.js App Router?**

Answer: Parallel routes (defined with `@slot-name` folder prefixes) let you render multiple independent route segments in the same layout simultaneously. The layout receives each slot as a named prop (`modal`, `sidebar`, etc.) alongside `children`. This enables patterns like: navigating to `/photos/42` shows a photo modal over the existing page (both the gallery and the modal are active routes simultaneously); or a dashboard layout where the main area and a side panel can navigate independently. It's the App Router equivalent of Vue Router's named views.

---

## Self-Assessment

- [ ] Explain what `<Outlet>` does and what happens if it's missing
- [ ] Describe how `layout.js` creates persistent layouts in Next.js App Router
- [ ] Implement a pathless layout route for authentication gating in React Router v6
- [ ] Explain what an index route is and why it's needed
- [ ] Describe the difference between nested routes and parallel routes

---
*Next: Route-level Code Splitting — how to load route components lazily to reduce initial bundle size.*
