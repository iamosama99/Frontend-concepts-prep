# BFF Pattern (Backend-for-Frontend)

## Quick Reference

| Concept | Mechanism | Implication |
|---|---|---|
| Per-client backend | A separate API server owned by each client team (web, iOS, Android) | Each client gets exactly the data shape it needs |
| Aggregation | BFF calls multiple upstream services, combines results | Client makes one request instead of N waterfall requests |
| Auth boundary | BFF holds session tokens; client holds only a session cookie | Tokens never touch the browser, reducing XSS attack surface |
| Team ownership | The frontend team owns the BFF | API contract changes happen without waiting for a shared backend team |

## What Is This?

The Backend-for-Frontend (BFF) pattern introduces a dedicated server-side layer between your frontend client and the downstream microservices or APIs. Rather than having every client talk directly to every service, each client surface (web, iOS, Android, TV) gets its own BFF — a lightweight server that speaks the client's language.

```
Without BFF:
  Web App → /users       (UserService)
  Web App → /orders      (OrderService)
  Web App → /products    (ProductService)
  ... three requests, a waterfall, over-fetching from generic APIs

With BFF:
  Web App → /dashboard   (Web BFF)
             ↓
  Web BFF → UserService, OrderService, ProductService (in parallel)
             ↓
  Returns exactly: { user: {name, avatar}, recentOrders: [...], recommended: [...] }
```

The BFF is not a monolith or a new microservice with business logic — it's a thin aggregation and transformation layer owned by the frontend team.

> **Check yourself:** A web app and a mobile app both need user profile data. The web app needs full address, payment methods, and order history. The mobile app only needs name, avatar, and notification preferences. Should they share one API endpoint? What's the problem if they do?

## Why Does It Exist?

### The Multi-Client Problem

When you have a single "generic" API serving multiple client types, you face an impossible design choice: build for the most demanding client (over-fetch for simpler clients) or build the minimal intersection (under-fetch for complex clients). Neither works well.

A web dashboard needs 15 fields from 3 services combined on one screen. A mobile notification widget needs 2 fields. A TV app needs a different subset optimized for large text. If all three consume the same API, you're constantly compromising.

### The Team Ownership Problem

Without a BFF, frontend teams depend on a shared backend team to change API responses. Want to reorganize how data is shaped for your new feature? File a ticket, wait for the backend team, coordinate a release. This tight coupling slows frontend development.

A BFF transfers API ownership to the frontend team. The team that builds the UI owns the API contract that feeds it. Changes to the web experience don't require backend team involvement.

### The Auth Security Problem

Modern web security best practices recommend keeping authentication tokens (JWTs, OAuth access tokens) off the browser. XSS can steal anything in JavaScript memory or localStorage. The BFF pattern solves this: the browser holds a simple HttpOnly session cookie (unreadable by JavaScript), and the BFF holds the actual OAuth tokens on the server side.

```
Browser → BFF (session cookie, httponly)
BFF → Upstream APIs (Bearer JWT token, stored in BFF memory/Redis)
```

The OAuth access token never touches browser JavaScript. Even a successful XSS attack can't steal the token — it's on the server.

## How It Works

### Request Aggregation

The BFF's primary job is combining multiple upstream calls into one client-facing response:

```javascript
// BFF route for the web dashboard page
app.get('/api/dashboard', authenticate, async (req, res) => {
  const userId = req.session.userId;

  // Parallel upstream calls — no waterfall
  const [user, orders, recommendations] = await Promise.all([
    userService.getProfile(userId),
    orderService.getRecentOrders(userId, { limit: 5 }),
    productService.getRecommendations(userId),
  ]);

  // Transform to exactly what the web UI needs
  res.json({
    user: { name: user.displayName, avatar: user.avatarUrl },
    orders: orders.map(o => ({
      id: o.id,
      date: o.createdAt,
      total: o.totalAmountCents / 100,
      status: o.fulfillmentStatus,
    })),
    recommended: recommendations.slice(0, 8).map(p => ({
      id: p.productId,
      name: p.displayName,
      price: p.priceCents / 100,
      image: p.images[0]?.url,
    })),
  });
});
```

The client makes one request. The BFF makes three requests in parallel. The response is shaped for the web UI's specific needs — not a generic canonical resource format.

### Shape Transformation

Upstream microservices often return "canonical" data shapes — complete, normalized, generic. The BFF transforms these into client-optimized shapes:

```javascript
// Upstream canonical User object (37 fields, all in snake_case, cents for money)
// Transformed to web-facing object (8 fields, camelCase, dollars for money)
const webUser = {
  id: upstreamUser.user_id,
  name: upstreamUser.display_name,
  email: upstreamUser.primary_email_address,
  avatarUrl: upstreamUser.profile_image_url,
  isPremium: upstreamUser.subscription_tier === 'premium',
  currency: upstreamUser.preferred_currency_code,
};
```

This transformation is valuable because: it keeps the client code clean, it hides upstream changes behind the BFF (upstream can rename `user_id` to `userId` without touching the client), and it enforces a stable contract at the BFF layer.

### Auth Token Handling

```javascript
// OAuth token exchange happens at the BFF — token never reaches the browser
app.get('/auth/callback', async (req, res) => {
  const { code } = req.query;

  // Exchange code for tokens on the server
  const tokens = await oauthProvider.exchangeCode(code);

  // Store access + refresh tokens in encrypted server session
  req.session.accessToken = encrypt(tokens.access_token);
  req.session.refreshToken = encrypt(tokens.refresh_token);

  // Set only a session ID cookie on the browser — httpOnly, Secure, SameSite
  // The browser's JavaScript never sees the actual OAuth token
  res.redirect('/dashboard');
});

// All authenticated requests: BFF reads token from session, adds to upstream calls
app.use(authenticate);
async function authenticate(req, res, next) {
  const accessToken = decrypt(req.session.accessToken);
  req.upstreamAuth = `Bearer ${accessToken}`;  // attached to upstream requests
  next();
}
```

## BFF vs. API Gateway

These are often confused but solve different problems:

| | API Gateway | BFF |
|---|---|---|
| Owned by | Platform/infra team | Frontend team |
| Purpose | Cross-cutting concerns (auth, rate limiting, routing) | Client-specific aggregation and transformation |
| Number | One (shared) | One per client surface |
| Business logic | None | Light transformation, no domain logic |
| Clients served | All clients | One specific client |

An API gateway sits in front of all services and handles common concerns — SSL termination, rate limiting, authentication, request logging. A BFF sits in front of one client and handles client-specific concerns. In many architectures both exist: API gateway → BFF → microservices.

## When to Use a BFF

**Use a BFF when:**
- You have multiple different client types (web, iOS, Android, TV) with significantly different data needs
- Frontend teams are blocked waiting for backend teams to reshape APIs
- You want to keep auth tokens off the browser
- Your clients make many waterfall requests that could be aggregated server-side
- You need different rate limits, caching strategies, or error handling per client surface

**Don't use a BFF when:**
- You have one client type — just call the API directly
- Your backend already has a GraphQL layer that clients query flexibly
- The BFF would just be a thin proxy with no transformation (adds latency with no value)
- Your team is small and the operational overhead of an extra service isn't worth it

## Gotchas

**BFF is not a place for business logic.** If the BFF is computing discounts, validating inventory, or making business decisions, you've made it a new microservice — and now it's a deployment dependency and reliability concern. Keep BFFs to aggregation and transformation only. Business logic lives in downstream services.

**BFF introduces another network hop.** Browser → BFF → upstream services adds one extra network hop. If the BFF is in the same region/datacenter as the upstream services, this internal hop is fast (< 1ms). If it's not co-located, you've added meaningful latency to every request. Co-locate BFFs with their upstream services.

**BFF caching must be designed deliberately.** A BFF response combines data from multiple sources. If one source has short TTL (live inventory) and another has long TTL (product descriptions), what's the BFF's cache policy? You need per-field or per-source caching logic rather than naively caching the aggregate response.

**Team ownership requires operational maturity.** If the frontend team owns the BFF, they need to be comfortable deploying, monitoring, and on-calling for a Node.js server. This is a meaningful operational responsibility that some frontend teams aren't ready for.

**BFF proliferation.** Once you create one BFF per client surface, you end up with a web BFF, a mobile BFF, a TV BFF, sometimes a desktop BFF, a partner API BFF. If these share significant logic, duplication becomes a maintenance problem. Shared libraries for common upstream client code, auth helpers, and error handling reduce duplication.

## Interview Questions

**Q (High): What problem does the BFF pattern solve, and what are its core responsibilities?**

Answer: The BFF pattern solves three related problems. First, the multi-client data shape problem: different clients (web, mobile, TV) need different data shapes from the same upstream services. A single generic API either over-fetches for simple clients or under-fetches for complex ones. A BFF per client surface returns exactly what each client needs. Second, the team coupling problem: frontend teams are blocked on backend teams to reshape API responses. With a BFF they own, frontend teams control the contract and iterate independently. Third, the auth security problem: OAuth tokens and session credentials can be held server-side in the BFF; the browser only holds a session cookie, keeping tokens out of browser JavaScript and therefore out of XSS reach. The BFF's responsibilities are aggregation (combining multiple upstream calls into one response) and transformation (reshaping canonical upstream data into client-optimized formats). It should not contain business logic.

The trap: Describing the BFF as "a backend that the frontend team owns" without explaining what it specifically does. The aggregation and transformation roles are the testable specifics.

---

**Q (High): How does a BFF improve the security of OAuth token handling compared to a pure SPA?**

Answer: In a pure SPA (no BFF), the OAuth access token must live somewhere in the browser — localStorage, sessionStorage, or memory. XSS exploits can read localStorage and sessionStorage. Memory tokens are safer but lost on page refresh. The BFF moves tokens entirely to the server: the OAuth code exchange happens on the BFF, the resulting access and refresh tokens are stored in an encrypted server-side session, and the browser receives only an HttpOnly session cookie (unreadable by JavaScript). When the browser makes requests to the BFF, it sends the session cookie automatically; the BFF attaches the appropriate OAuth token to upstream service calls. The browser's JavaScript layer can never access the OAuth token — even a complete XSS compromise of the page can't steal it. This makes the BFF pattern the recommended approach for SPAs that handle sensitive data with OAuth authentication.

The trap: Not knowing that HttpOnly cookies are inaccessible to JavaScript. This is the key security property that makes the BFF approach superior.

---

**Q (Medium): What is the difference between a BFF and an API Gateway?**

Answer: An API gateway is a shared platform-level component that handles cross-cutting concerns for all services and all clients: SSL termination, rate limiting, authentication validation, request/response logging, routing. One gateway serves everything. A BFF is client-specific: it's a dedicated server for one client surface (web, mobile, etc.), handling aggregation of multiple upstream calls and transformation of responses into client-optimized shapes. One BFF per client. In a production architecture, both often exist in sequence: browser → API gateway (auth check, rate limit) → Web BFF (aggregate, transform) → microservices. The key distinction is ownership and purpose: gateway is owned by the infrastructure/platform team and is shared; BFF is owned by the frontend team and is client-specific.

The trap: Conflating the two into "they're both proxies." The ownership model (shared platform vs. per-team) and purpose (cross-cutting concerns vs. client-specific aggregation) are the real distinction.

---

**Q (Medium): When is a BFF not the right choice?**

Answer: A BFF adds operational overhead — another service to deploy, monitor, scale, and on-call for. It's not worth it when: (1) You have a single client type — the BFF adds a network hop with no multi-client benefit. (2) You're already using GraphQL with a flexible schema — the client can query exactly the data it needs without a BFF doing the reshaping. (3) Your backend team is small and responsive enough that frontend teams aren't blocked. (4) The transformation would be trivial — if the BFF is just passing requests through with no aggregation or transformation, it adds latency without value. (5) The team doesn't have the operational maturity to own and operate a server. The BFF pattern solves a real problem at sufficient scale and team size; under that threshold, it's premature complexity.

The trap: Recommending a BFF for every architecture without acknowledging the operational cost.

---

**Q (Low): How does BFF caching work when aggregating data with different freshness requirements?**

Answer: When a BFF aggregates data from multiple upstream sources with different freshness requirements (live inventory that updates per-second, product descriptions that update daily), the naive approach of caching the combined response is wrong — you'd either over-cache live data or under-cache stable data. Correct approaches: (1) Cache upstream responses individually by their appropriate TTL before aggregating — product description cached for 1 hour, inventory cached for 10 seconds; (2) Return appropriate `Cache-Control` headers from the BFF reflecting the minimum TTL of all constituent data (the lowest freshness requirement drives the overall policy); (3) For partially dynamic responses, separate the stable parts (cached) from the live parts (not cached) and have the client compose them, or use micro-frontend techniques to compose independently-cached sections. In practice, many BFF responses mix static and live data and aren't cached at the HTTP layer — the BFF's value is aggregation and team ownership, not caching.

The trap: Thinking BFF caching is straightforward. The complexity of aggregating data with different TTLs is a genuine design challenge.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain the three problems BFF solves: multi-client shapes, team coupling, auth security
- [ ] Can describe what a BFF actually does: aggregation + transformation, not business logic
- [ ] Can explain how BFF improves OAuth token security (tokens on server, only session cookie to browser)
- [ ] Can distinguish BFF from API gateway by ownership and purpose
- [ ] Can name two scenarios where a BFF is not the right choice
- [ ] Can explain why BFF caching is non-trivial when sources have different freshness requirements

---
*Next: Data Fetching Strategies — SWR, React Query, and Optimistic UI — the client-side patterns for managing server state, caching, and making mutations feel instant.*
