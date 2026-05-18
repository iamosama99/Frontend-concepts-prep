# Logging, Tracing & Session Replay

## Quick Reference

| Tool | What It Captures | Primary Use |
|---|---|---|
| Structured logging | Discrete events with metadata | Debugging what happened |
| Distributed tracing | Request spans across services | Debugging why it was slow |
| Session replay | Video-like user interaction recording | Reproducing UI bugs |
| OpenTelemetry | Vendor-neutral telemetry SDK | Portable instrumentation |

---

## Why Frontend Observability is Hard

Server logs record every request. Browser JavaScript runs in a user's device — you see nothing unless you explicitly collect and send it. A user reports "the checkout button didn't work" and you have no information about what happened in their browser.

Frontend observability is about answering: what happened, in what sequence, in what context, for which user? The three pillars — logs, traces, and session replay — each answer a different part of this.

---

## Structured Logging

Unstructured logs (`console.log('error: something broke')`) are useless at scale — you can't query, filter, or aggregate them. **Structured logging** emits JSON with consistent fields, making logs queryable.

**Basic structured logger:**

```typescript
type LogLevel = 'debug' | 'info' | 'warn' | 'error';

interface LogEntry {
  level: LogLevel;
  message: string;
  timestamp: number;
  url: string;
  sessionId: string;
  userId?: string;
  [key: string]: unknown;
}

class Logger {
  private sessionId: string;
  private context: Record<string, unknown> = {};

  constructor(sessionId: string) {
    this.sessionId = sessionId;
  }

  setContext(context: Record<string, unknown>): void {
    this.context = { ...this.context, ...context };
  }

  private emit(level: LogLevel, message: string, extra: Record<string, unknown> = {}): void {
    const entry: LogEntry = {
      level,
      message,
      timestamp: Date.now(),
      url: window.location.pathname,
      sessionId: this.sessionId,
      ...this.context,
      ...extra,
    };

    // Batch and send to logging endpoint
    logQueue.push(entry);
    scheduleFlush();

    // Also log to console in development
    if (process.env.NODE_ENV === 'development') {
      console[level](message, entry);
    }
  }

  debug(message: string, extra?: Record<string, unknown>): void { this.emit('debug', message, extra); }
  info(message: string, extra?: Record<string, unknown>): void { this.emit('info', message, extra); }
  warn(message: string, extra?: Record<string, unknown>): void { this.emit('warn', message, extra); }
  error(message: string, extra?: Record<string, unknown>): void { this.emit('error', message, extra); }
}

const logger = new Logger(generateSessionId());

// Usage — rich context makes logs queryable
logger.setContext({ userId: '123', plan: 'pro', featureFlag: 'new-checkout' });
logger.info('checkout_started', { cartItems: 3, totalValue: 149.99 });
logger.error('payment_failed', { provider: 'stripe', errorCode: 'card_declined' });
```

**What structured logs enable:**

```
Query: level=error AND url=/checkout AND timestamp > 1h ago
Query: event=payment_failed AND provider=stripe GROUP BY errorCode
Query: sessionId=abc123  → all events for that session in order
```

---

## Sentry — Error Tracking with Context

Sentry is the standard error tracking tool. It captures exceptions, enriches them with context (breadcrumbs, user info, release version), and groups similar errors automatically.

```typescript
import * as Sentry from '@sentry/react';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.COMMIT_SHA, // link errors to specific deploys
  tracesSampleRate: 0.1, // sample 10% of transactions for performance
  integrations: [
    new Sentry.BrowserTracing(), // automatic page load and navigation traces
  ],
});

// Set user context (appears on every error)
Sentry.setUser({ id: user.id, email: user.email });

// Add breadcrumbs (ordered trail of what happened before the error)
Sentry.addBreadcrumb({
  message: 'User clicked checkout',
  level: 'info',
  data: { cartItems: 3 },
});

// Capture a manual exception
try {
  await processPayment(cart);
} catch (error) {
  Sentry.captureException(error, {
    extra: { cartId: cart.id, userId: user.id },
    tags: { feature: 'checkout', paymentProvider: 'stripe' },
  });
  throw error; // re-throw so the UI can handle it
}
```

---

## Distributed Tracing with OpenTelemetry

Distributed tracing tracks a single user action as it propagates through multiple services. A frontend trace starts in the browser and can be linked to backend spans, making it possible to see the full latency chain: browser → CDN → API gateway → database.

**OpenTelemetry** is the vendor-neutral SDK for emitting traces, metrics, and logs. It separates instrumentation (in your code) from the backend (Datadog, Jaeger, Honeycomb, etc.).

```typescript
import { trace, context, propagation } from '@opentelemetry/api';
import { WebTracerProvider } from '@opentelemetry/sdk-trace-web';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const provider = new WebTracerProvider();
provider.addSpanProcessor(new BatchSpanProcessor(
  new OTLPTraceExporter({ url: '/otel-collector/traces' })
));
provider.register();

const tracer = trace.getTracer('frontend', '1.0.0');

// Instrument a user action as a trace
async function submitOrder(cart: Cart): Promise<Order> {
  return tracer.startActiveSpan('submitOrder', async (span) => {
    span.setAttributes({
      'cart.item_count': cart.items.length,
      'cart.total': cart.total,
      'user.id': currentUser.id,
    });

    try {
      // Propagate trace context to the backend via HTTP headers
      const headers: Record<string, string> = {};
      propagation.inject(context.active(), headers);

      const response = await fetch('/api/orders', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', ...headers },
        body: JSON.stringify(cart),
      });

      const order = await response.json();
      span.setAttributes({ 'order.id': order.id });
      return order;
    } catch (error) {
      span.recordException(error as Error);
      span.setStatus({ code: SpanStatusCode.ERROR });
      throw error;
    } finally {
      span.end();
    }
  });
}
```

The `propagation.inject` call adds trace context headers (`traceparent`, `tracestate`) to the HTTP request. When the backend receives these headers and creates its own spans, those spans are linked to the frontend span — you see the full call tree from user click to database query.

---

## Session Replay (LogRocket, FullStory, Sentry)

Session replay records user interactions — mouse movements, clicks, scroll, form inputs — and reconstructs a "video" of what the user saw. Unlike a video recording, it typically works by recording DOM mutations (MutationObserver) and replaying them, not by capturing pixel frames. This makes it storage-efficient and privacy-configurable.

**What it enables:**
- See the exact UI state when a bug was reported
- Understand user confusion without asking them to reproduce it
- Link a Sentry error to the session replay that triggered it

**Privacy controls — essential:**

```typescript
import LogRocket from 'logrocket';

LogRocket.init('org/project', {
  dom: {
    // Mask sensitive inputs by default (passwords, credit cards)
    inputSanitizer: true,
    // Mask specific elements
    privateAttributeBlocklist: ['data-sensitive'],
  },
  network: {
    // Redact auth headers from recorded network requests
    requestSanitizer: (request) => {
      if (request.headers['Authorization']) {
        request.headers['Authorization'] = '[REDACTED]';
      }
      return request;
    },
  },
});

// Identify the user (links session to a user ID, not PII)
LogRocket.identify(user.id, {
  plan: user.plan,
  // Don't record email/name in replay tools unless you have explicit consent
});

// Link LogRocket session to Sentry error
LogRocket.getSessionURL(sessionUrl => {
  Sentry.configureScope(scope => {
    scope.setExtra('sessionReplayURL', sessionUrl);
  });
});
```

**When to use session replay:**
- Bug reports with vague descriptions ("it just stopped working")
- UX research — understand drop-off points in a funnel
- Support tickets — see exactly what the user did

**When NOT to use session replay:**
- Without clear privacy policy disclosure to users
- On pages with highly sensitive data (banking, medical) without aggressive masking
- As a primary debugging tool (it's expensive on storage at high volumes — sample it)

---

## What to Instrument

A practical instrumentation checklist:

```typescript
// 1. Route changes — know where users go
router.on('routeChange', (to) => logger.info('route_change', { path: to }));

// 2. API errors — know when fetches fail
async function apiFetch(url: string, options?: RequestInit): Promise<Response> {
  const res = await fetch(url, options);
  if (!res.ok) {
    logger.error('api_error', { url, status: res.status, statusText: res.statusText });
    Sentry.captureMessage(`API error: ${res.status} ${url}`, 'error');
  }
  return res;
}

// 3. Key user actions — know what the user did
function trackEvent(name: string, properties?: Record<string, unknown>): void {
  logger.info('user_event', { event: name, ...properties });
  analytics.track(name, properties); // also send to product analytics
}

// 4. Performance marks — know how long key flows take
performance.mark('checkout:payment:start');
await processPayment();
performance.mark('checkout:payment:end');
performance.measure('payment_duration', 'checkout:payment:start', 'checkout:payment:end');
```

---

> **Check yourself:** A user reports that clicking "Submit" on the checkout page did nothing. What would you look for in each of logs, traces, and session replay?

---

## Self-Assessment

- [ ] I understand why structured JSON logs are needed over plain `console.log`
- [ ] I know what Sentry captures (errors + breadcrumbs) and how to add context
- [ ] I understand what a distributed trace is and how trace context propagates from browser to backend
- [ ] I know how session replay works (DOM mutations, not video) and its privacy implications
- [ ] I know which user interactions and errors to instrument
- [ ] I understand what OpenTelemetry is and why being vendor-neutral matters

---

## Interview Q&A

**Q: What is structured logging and why does it matter at scale?** `High`

Structured logging emits log entries as JSON objects with consistent, named fields — timestamp, level, message, user ID, URL, etc. — rather than free-form strings. At scale, this makes logs queryable: you can filter by field values, aggregate counts, and join across events. Unstructured strings like `"Error: user 123 payment failed"` require regex parsing and break as the format changes. Structured logs (`{ event: 'payment_failed', userId: '123', provider: 'stripe' }`) remain queryable even as you add fields.

---

**Q: What is distributed tracing and how does it work across the browser/backend boundary?** `High`

Distributed tracing tracks a single user request as it flows through multiple services. Each service creates "spans" — time-boxed units of work — that share a common `traceId`. In the browser, a span is started for the user action (e.g., submitting an order). When the browser makes an HTTP request, trace context is injected into headers (`traceparent`). The backend reads those headers and creates child spans with the same `traceId`. The result is a complete call tree: browser action → API call → database query — with timing for each step. This makes it possible to diagnose whether a slowdown is frontend rendering, network latency, or a slow database query.

---

**Q: How does session replay work technically, and what are the privacy concerns?** `Medium`

Session replay tools use `MutationObserver` to record DOM changes, mouse movements, clicks, and scroll events, then serialize them into a structured format. Playback reconstructs the DOM state over time — it's effectively a recording of DOM mutations, not a pixel video. This is storage-efficient but requires privacy controls: form inputs can contain passwords or PII, so reputable tools mask all input fields by default. You should also redact sensitive network requests and not record pages with highly sensitive data (medical, financial) without aggressive masking and user consent. GDPR and similar regulations require disclosure of session recording in your privacy policy.

---

**Q: What is OpenTelemetry and why is it valuable?** `Medium`

OpenTelemetry is a vendor-neutral, open-source SDK and protocol for collecting traces, metrics, and logs. "Vendor-neutral" is the key property: you instrument your code once using the OTel API, and you can send the data to any backend (Datadog, Honeycomb, Jaeger, etc.) by changing your exporter configuration. Without OTel, you'd use vendor-specific SDKs, creating vendor lock-in at the instrumentation layer. Migrating backends would require re-instrumenting all your code.

---

**Q: What should you prioritize instrumenting on the frontend?** `Low`

In rough priority order: error boundaries and global unhandledrejection (catch what breaks), API request failures with status codes and URLs (catch what's slow or failing), key user actions in important flows (checkout, login, core product actions), route changes (navigation analytics and timing), and Core Web Vitals (LCP, INP, CLS for field performance data). The goal is to be able to reconstruct the sequence of events that led to any bug report. Instrument the decision points and failure surfaces, not every user click.
