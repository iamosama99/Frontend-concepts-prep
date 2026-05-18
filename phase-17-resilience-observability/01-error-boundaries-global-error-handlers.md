# Error Boundaries & Global Error Handlers

## Quick Reference

| Mechanism | Catches | Does Not Catch | Where |
|---|---|---|---|
| React Error Boundary | Render errors, lifecycle errors, constructor errors | Async errors, event handlers, SSR | Component tree |
| `window.onerror` | Uncaught synchronous errors, unhandled promise rejections (older) | Errors already caught by try/catch | Global |
| `window.addEventListener('unhandledrejection')` | Unhandled Promise rejections | Handled rejections | Global |
| `try/catch` in async functions | Errors inside the try block | Errors in other async chains | Explicit |

---

## Why Error Boundaries Exist

Without error boundaries, an unhandled render error in any component tears down the **entire** React tree — the user sees a blank page. Error boundaries catch render errors in their subtree and display fallback UI instead, keeping the rest of the application functional.

The problem they solve: a component deep in the tree (say, a third-party chart widget) throws during render. Without a boundary, React unmounts everything. With a boundary around the chart section, only that section shows the fallback; the navigation, sidebar, and main content survive.

---

## React Error Boundary — Class Component

Error boundaries must be class components — there is no Hook equivalent for the two required methods (`getDerivedStateFromError` and `componentDidCatch`):

```typescript
import { Component, ErrorInfo, ReactNode } from 'react';

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

interface ErrorBoundaryProps {
  children: ReactNode;
  fallback?: ReactNode | ((error: Error) => ReactNode);
  onError?: (error: Error, info: ErrorInfo) => void;
}

class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  state: ErrorBoundaryState = { hasError: false, error: null };

  // Called during render to update state when an error occurs
  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  // Called after render — use for logging, not for state updates
  componentDidCatch(error: Error, info: ErrorInfo): void {
    this.props.onError?.(error, info);
    // info.componentStack: the React component stack trace
    console.error('Error boundary caught:', error, info.componentStack);
  }

  render(): ReactNode {
    if (this.state.hasError && this.state.error) {
      const { fallback } = this.props;
      if (typeof fallback === 'function') {
        return fallback(this.state.error);
      }
      return fallback ?? <div>Something went wrong.</div>;
    }
    return this.props.children;
  }
}
```

**Usage:**

```typescript
// Granular boundaries — only wrap sections that might fail independently
function Dashboard() {
  return (
    <div>
      <Navigation /> {/* not wrapped — must never fail */}
      <ErrorBoundary fallback={<ChartError />} onError={reportToSentry}>
        <RevenueChart /> {/* isolated — can fail without breaking the page */}
      </ErrorBoundary>
      <ErrorBoundary fallback={<TableError />} onError={reportToSentry}>
        <DataTable />
      </ErrorBoundary>
    </div>
  );
}
```

**What error boundaries do NOT catch:**

```typescript
// Not caught: error in an event handler
function Button() {
  return (
    <button onClick={() => {
      throw new Error('click error'); // NOT caught by error boundary
    }}>
      Click
    </button>
  );
}

// Not caught: async error
function Component() {
  useEffect(() => {
    fetchData().then(() => {
      throw new Error('async error'); // NOT caught by error boundary
    });
  }, []);
  return <div />;
}
```

Event handler and async errors must be caught with `try/catch` and fed into component state, or caught by global handlers.

---

## Reset an Error Boundary

A key UX consideration: give users a way to retry after a non-fatal error.

```typescript
interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
  resetKey: number;
}

class ResettableErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  state = { hasError: false, error: null, resetKey: 0 };

  static getDerivedStateFromError(error: Error): Partial<ErrorBoundaryState> {
    return { hasError: true, error };
  }

  reset = (): void => {
    this.setState({ hasError: false, error: null, resetKey: this.state.resetKey + 1 });
  };

  render(): ReactNode {
    if (this.state.hasError) {
      return (
        <div>
          <p>Something went wrong.</p>
          <button onClick={this.reset}>Try again</button>
        </div>
      );
    }
    return <div key={this.state.resetKey}>{this.props.children}</div>;
  }
}
```

The `key` on the children div forces React to remount the subtree on reset, clearing any lingering error state in child components.

---

## Global Error Handlers

**`window.onerror` — synchronous errors and script errors:**

```typescript
window.onerror = (
  message: string | Event,
  source: string | undefined,
  lineno: number | undefined,
  colno: number | undefined,
  error: Error | undefined
): boolean => {
  reportError({
    message: typeof message === 'string' ? message : message.type,
    source,
    lineno,
    colno,
    stack: error?.stack,
  });

  // Return true to suppress the browser's default error logging.
  // Return false (or omit) to let the default behavior run in addition.
  return false;
};
```

**`unhandledrejection` — unhandled Promise rejections:**

```typescript
window.addEventListener('unhandledrejection', (event: PromiseRejectionEvent) => {
  event.preventDefault(); // suppress default console error

  const reason = event.reason;
  reportError({
    type: 'UnhandledRejection',
    message: reason instanceof Error ? reason.message : String(reason),
    stack: reason instanceof Error ? reason.stack : undefined,
  });
});
```

**`rejectionhandled` — late-handled rejections:**

```typescript
// Fires when a Promise rejection is eventually handled after being initially unhandled
window.addEventListener('rejectionhandled', (event: PromiseRejectionEvent) => {
  // Remove from pending error reports if you were about to report it
});
```

---

## Error Reporting Pattern

A complete error reporting utility:

```typescript
interface ErrorReport {
  message: string;
  stack?: string;
  context?: Record<string, unknown>;
  severity: 'fatal' | 'error' | 'warning';
}

class ErrorReporter {
  private queue: ErrorReport[] = [];
  private isFlushing = false;

  report(report: ErrorReport): void {
    this.queue.push({
      ...report,
      context: {
        url: window.location.href,
        userAgent: navigator.userAgent,
        timestamp: Date.now(),
        ...report.context,
      },
    });
    this.scheduleFlush();
  }

  private scheduleFlush(): void {
    if (this.isFlushing) return;
    this.isFlushing = true;
    // Batch reports: wait for current call stack to unwind
    queueMicrotask(() => this.flush());
  }

  private flush(): void {
    if (this.queue.length === 0) { this.isFlushing = false; return; }
    const batch = this.queue.splice(0);
    this.isFlushing = false;

    navigator.sendBeacon('/api/errors', JSON.stringify(batch));
    // sendBeacon works even if the page is closing
  }
}

const reporter = new ErrorReporter();

// Wire up global handlers
window.addEventListener('unhandledrejection', (e) => {
  reporter.report({
    message: e.reason?.message ?? String(e.reason),
    stack: e.reason?.stack,
    severity: 'error',
  });
});
```

---

> **Check yourself:** An async function inside `useEffect` throws. Does the nearest React error boundary catch it? How do you handle it instead?

---

## Self-Assessment

- [ ] I understand why error boundaries must be class components
- [ ] I know what error boundaries do and do not catch
- [ ] I know how to reset an error boundary to let users retry
- [ ] I know the difference between `window.onerror` and `unhandledrejection`
- [ ] I can implement a global error reporting pipeline
- [ ] I understand that `navigator.sendBeacon` is used for reliable error reporting on page close

---

## Interview Q&A

**Q: What is a React error boundary and what does it catch?** `High`

A React error boundary is a class component that implements `getDerivedStateFromError` and/or `componentDidCatch`. It catches errors that occur during rendering, in lifecycle methods, and in constructors of child components — replacing the crashed subtree with fallback UI instead of taking down the entire app. It does NOT catch errors in event handlers (those need explicit `try/catch`), asynchronous code (Promise chains, `setTimeout`), or server-side rendering.

---

**Q: Why can't you write a React error boundary as a function component?** `High`

Error boundaries rely on two class lifecycle methods: `getDerivedStateFromError` (which runs during the render phase to update state) and `componentDidCatch` (which runs after the render phase for side effects like logging). There is no Hook equivalent for these — `useEffect` and `useState` don't map to the render-phase interception that `getDerivedStateFromError` requires. The React team has stated they plan to add this capability to function components eventually, but as of React 18 it remains class-only. In practice, you write one reusable class-based boundary and wrap it around all your function components.

---

**Q: How does `window.onerror` differ from `window.addEventListener('unhandledrejection')`?** `Medium`

`window.onerror` catches uncaught synchronous JavaScript errors — exceptions thrown in synchronous code that nothing else caught. It also catches script loading errors. `unhandledrejection` catches Promise rejections that have no `.catch()` handler attached — async errors. They're complementary: a try/catch-less async function that throws will trigger `unhandledrejection` but not `onerror`. A synchronous throw outside a try/catch triggers `onerror` but not `unhandledrejection`. A complete global error handling setup subscribes to both.

---

**Q: How do you give users a way to retry after an error boundary catches an error?** `Medium`

You provide a "Try again" button in the fallback UI that calls a reset method on the boundary. The reset method sets `hasError` to `false` and increments a key. The key change forces React to remount the children subtree (rather than reuse the errored instances), clearing any lingering error state. Without the remount via key change, calling `setState({ hasError: false })` would re-render the same errored subtree, which might immediately error again.

---

**Q: Why use `navigator.sendBeacon` for error reporting instead of `fetch`?** `Low`

`sendBeacon` sends data to a server asynchronously without blocking the page and — critically — it completes even if the page is being closed (navigating away, tab closing, page reload). `fetch` requests may be cancelled if the page unloads before the request completes. Since errors often trigger as the user navigates or closes the page, `sendBeacon` ensures the error report actually reaches the server. The downside is that you can't read the response from `sendBeacon`, but error reporting is fire-and-forget by nature.
