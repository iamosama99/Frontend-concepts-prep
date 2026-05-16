# API Paradigms: REST, GraphQL, gRPC-web, tRPC

## Quick Reference

| Paradigm | Transport | Schema | Over-fetching | Best fit |
|---|---|---|---|---|
| REST | HTTP/1.1+ | None (convention) | Common | Public APIs, simple CRUD, cross-team contracts |
| GraphQL | HTTP/1.1+ | SDL (typed) | Eliminated | Client-driven data needs, multiple client types |
| gRPC-web | HTTP/2 | Protobuf IDL (binary) | Minimal | High-throughput internal services, typed contracts |
| tRPC | HTTP/1.1+ | TypeScript types | Configurable | Full-stack TypeScript monorepos, end-to-end type safety |

## What Is This?

These are four different approaches to structuring the contract between a frontend and a backend. Each makes different trade-offs on where schema lives, who decides the shape of data, and how the request/response cycle works.

They all move data between client and server — the question is who controls the shape of that data and what guarantees they make to each other.

> **Check yourself:** REST has no built-in schema. GraphQL has a schema. What does having a schema actually buy you that REST doesn't provide automatically?

## REST

REST (Representational State Transfer) is an architectural style, not a protocol. A REST API expresses resources as URLs and uses HTTP methods to express operations:

```
GET    /users/42          → read user 42
POST   /users            → create a user
PUT    /users/42          → replace user 42
PATCH  /users/42          → partial update user 42
DELETE /users/42          → delete user 42
```

Each URL is a resource. The HTTP method is the verb. The response is whatever the server decides to return — there's no spec for the response shape, only convention.

**Over-fetching:** The server decides what a resource looks like. `GET /users/42` returns every field the server deems appropriate — name, email, address, preferences, avatar — even if the client only needs the name. This wastes bandwidth and inflates the client's parsing work.

**Under-fetching:** One resource often isn't enough. A dashboard that needs user + their orders + their preferences makes three sequential requests (`/users/42`, `/users/42/orders`, `/users/42/preferences`), creating a waterfall.

**No schema enforcement:** REST endpoints return whatever JSON they want. If the backend changes a field name, the frontend breaks at runtime — there's no build-time contract. OpenAPI/Swagger adds schema on top of REST, but it's opt-in and easily out of sync.

**Caching:** REST's URL-per-resource model aligns naturally with HTTP caching. `GET /products/123` can be CDN-cached indefinitely. This is a genuine structural advantage over GraphQL.

```javascript
// Standard REST fetch
const user = await fetch('/api/users/42').then(r => r.json());
// Returns everything — name, email, address, preferences, avatar...
// Even if you only needed the name
```

## GraphQL

GraphQL is a query language for APIs. The client specifies exactly what data it needs in the request body; the server returns exactly that.

```graphql
# Client sends this query
query {
  user(id: "42") {
    name
    email
    orders(last: 5) {
      id
      total
      status
    }
  }
}
```

The server returns `{ user: { name: "...", email: "...", orders: [...] } }` — nothing more, nothing less.

**Schema:** GraphQL has a strongly typed schema defined in SDL (Schema Definition Language). Every type, field, and relationship is declared. The schema is the contract — if a field doesn't exist in the schema, it can't be queried. This enables introspection (tools can discover the full API shape) and codegen (generate TypeScript types from the schema automatically).

**Solves over-fetching and under-fetching:** One request, exactly the data the client needs, across any number of related resources.

**N+1 problem:** The server-side challenge of GraphQL. If a query requests 100 users and each user's orders, a naive resolver fires 100 separate database queries for orders. The DataLoader pattern batches and caches these.

**Caching complexity:** GraphQL endpoints are typically a single URL (`POST /graphql`). Identical queries return identical data, but HTTP cache by URL doesn't work — everything goes to the same endpoint. Client-side normalized caches (Apollo, urql) cache by entity identity (`User:42`) across queries, but CDN caching requires persisted queries or GET-based queries with query in the URL.

**Single endpoint:** One URL, all queries. Simplifies routing but concentrates load and complicates request-level access control (every query hits the same endpoint).

```javascript
// GraphQL — client controls the shape
const { user } = await gqlClient.query({
  query: gql`
    query GetUser($id: ID!) {
      user(id: $id) { name email }  # exactly these fields
    }
  `,
  variables: { id: '42' }
});
```

## gRPC-web

gRPC uses Protocol Buffers (protobuf) — a binary serialization format — and HTTP/2 for transport. It's primarily a backend-to-backend technology; `gRPC-web` is a browser-compatible variant that works over HTTP/1.1 and HTTP/2 with a proxy.

**Contract-first:** A `.proto` file defines services and message types. Code is generated for both client and server from this definition. The generated code is strongly typed in every language.

```protobuf
// user.proto
service UserService {
  rpc GetUser (UserRequest) returns (User);
  rpc ListUsers (ListRequest) returns (stream User);
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
}
```

**Binary encoding:** Protobuf is significantly more compact than JSON (~3–10x smaller for equivalent data) and faster to serialize/deserialize. For high-frequency data or large payloads, this is a real performance win.

**Streaming:** gRPC natively supports server streaming, client streaming, and bidirectional streaming over HTTP/2. A subscription to live events is natural to express in gRPC.

**Browser friction:** Browsers can't speak raw gRPC — they require an Envoy proxy or gRPC-web gateway to translate. This adds infrastructure complexity. For browser clients, this overhead often makes GraphQL or tRPC more practical choices.

**Best suited for:** Internal service-to-service communication where payload efficiency and strict typing matter, with a separate GraphQL or REST gateway for browser clients.

## tRPC

tRPC is a TypeScript-first RPC framework that eliminates the API layer entirely for full-stack TypeScript projects. The server defines procedures in TypeScript, and the client calls them with full type inference — no schema language, no codegen step, no serialization format to define.

```typescript
// Server (Next.js API route or Express)
const appRouter = router({
  getUser: procedure
    .input(z.object({ id: z.string() }))
    .query(async ({ input }) => {
      return db.users.findById(input.id);
      // Return type is inferred — TypeScript knows the shape
    }),
});

// Client — no fetch, no query string, just a typed function call
const user = await trpc.getUser.query({ id: '42' });
//     ↑ fully typed — user has the exact type the server returns
```

**End-to-end type safety without codegen:** TypeScript's type inference flows from server procedure to client call. If the server changes the return type, the client immediately shows type errors. No schema to write, no code to generate, no sync step.

**Zero extra infrastructure:** tRPC uses HTTP under the hood (GET for queries, POST for mutations) but abstracts it entirely. No GraphQL gateway, no protobuf compiler, no API versioning.

**Constraint:** Both client and server must be TypeScript. This works naturally in a Next.js full-stack app or a monorepo where frontend and backend share a codebase. It falls apart for public APIs (you can't give external consumers TypeScript types as a contract) or polyglot backends.

**Not a replacement for GraphQL at scale:** tRPC works best when there's one backend serving one frontend. For multiple frontends (web, iOS, Android) consuming the same API, or for public APIs with external consumers, a formal schema (GraphQL SDL or OpenAPI) is more appropriate.

> **Check yourself:** A startup is building a full-stack Next.js app with one team owning both frontend and backend. Another startup is building a public API that iOS, Android, and web apps will all consume. Which paradigm suits each?

## Choosing Between Them

| Scenario | Recommendation |
|---|---|
| Public API with external consumers | REST + OpenAPI or GraphQL |
| Mobile + web + third-party all consuming one API | GraphQL (flexible per-client queries) |
| Full-stack TypeScript, one team | tRPC (no boilerplate, perfect types) |
| Microservice-to-microservice, high throughput | gRPC |
| Simple CRUD, small team, CDN caching matters | REST |
| Existing REST API, add flexibility | GraphQL layer over REST resolvers |

## Gotchas

**REST "just works" until it doesn't.** Teams underestimate REST's pain until the product has 5+ client types (web, iOS, Android, TV, smartwatch) each needing different data shapes. At that scale, over-fetching and waterfall requests compound into a real performance problem that GraphQL was designed to solve.

**GraphQL is not automatically more performant.** The flexibility that eliminates over-fetching also eliminates server-side caching simplicity. A naive GraphQL server with complex nested queries can perform worse than a simple REST endpoint that returns cached JSON.

**tRPC's type safety has a build-time dependency.** The types only flow if the client is consuming the actual server router type. If your deployment architecture separates the frontend and backend builds, you need to share the router type via a package — this works but adds complexity in non-monorepo setups.

**gRPC-web requires a proxy in production.** Browsers can't speak raw HTTP/2 gRPC. Envoy proxy, gRPC-web shims, or Connect-web (a modern alternative from Buf) are needed. Teams often underestimate this infrastructure cost.

**GraphQL N+1 is a serious production problem.** Every nested relationship in a GraphQL query is a potential N+1 query pattern. Without DataLoader, a query for 100 posts with their authors fires 101 database queries (1 for posts, 100 for authors). This is the top production pitfall in GraphQL backends and must be addressed before going to production.

## Interview Questions

**Q (High): What are over-fetching and under-fetching in REST, and how does GraphQL address them?**

Answer: Over-fetching is when a REST endpoint returns more data than the client needs. `GET /users/42` returns the full user object — address, preferences, avatar, audit metadata — when the client only needs the name and email. This wastes bandwidth and parsing time. Under-fetching is when a single endpoint doesn't provide enough data, forcing multiple round-trips. A profile page needing user + their posts + their follower count requires three sequential requests: `/users/42`, `/users/42/posts`, `/users/42/stats`. Sequential requests create a waterfall. GraphQL addresses both: the client specifies exactly which fields to return in a single query, and can fetch related resources in one round-trip. The server returns exactly what was requested — no more, no less. This is the fundamental value proposition of GraphQL.

The trap: Stopping at "GraphQL is faster." The question is about the specific mechanisms. Name the problems, name how GraphQL resolves each one.

---

**Q (High): Why is caching harder with GraphQL than REST, and what are the solutions?**

Answer: REST's URL-per-resource model aligns with HTTP caching — `GET /products/123` is a stable URL that CDNs and browsers can cache by URL + headers. GraphQL typically uses a single endpoint (`POST /graphql`), so HTTP cache-by-URL can't distinguish between different queries to the same endpoint. POST requests aren't cached by default. Solutions: (1) Persisted queries — client sends a hash of the query instead of the full query text, server looks up the query by hash; the hash becomes a stable cache key for CDN caching. (2) GET-based queries — send simple queries as GET with query in the URL parameter; cacheable by CDN, but query string length limits apply. (3) Client-side normalized caching (Apollo Client, urql) — cache results by entity identity (`User:42`) rather than by URL; entities are shared across queries automatically. (4) Response caching at the field level using `@cacheControl` directives in some GraphQL servers.

The trap: "GraphQL can't be cached." It can — it just requires different strategies. Name at least two solutions.

---

**Q (High): What is tRPC and what constraint makes it unsuitable for public APIs?**

Answer: tRPC is a TypeScript RPC framework that achieves end-to-end type safety without a schema language or code generation step. The server defines procedures as TypeScript functions; TypeScript's type inference flows from the server return types directly to the client call sites. If the server changes a return type, the client immediately shows type errors. The constraint: both client and server must be TypeScript in the same type system — the client imports the server's router type. This works perfectly in a monorepo or a Next.js full-stack project. It breaks for public APIs because external consumers (mobile apps, third-party developers, non-TypeScript services) can't import TypeScript types. Public APIs need a language-agnostic schema — OpenAPI for REST or GraphQL SDL — that can generate clients for any language. tRPC is the right choice for internal full-stack TypeScript products; GraphQL or REST is the right choice for public APIs or multi-language backends.

The trap: Describing tRPC without naming why it can't work for external consumers. The constraint is the "TypeScript-only" boundary.

---

**Q (Medium): What is the N+1 problem in GraphQL and how does DataLoader solve it?**

Answer: The N+1 problem occurs when resolving a list of items with a related field. A query for 100 posts with their author's name triggers: 1 query for all posts, then 100 individual queries for each author — 101 total. This happens because each post's `author` resolver fires independently, unaware of the other resolvers' intent. DataLoader solves this with batching and caching. Instead of immediately fetching each author, DataLoader collects all author ID requests within the same event loop tick, then fires a single batched query (`SELECT * FROM users WHERE id IN (1, 2, 3, ...)`) and distributes results back to the individual resolvers. The result: 2 total queries regardless of list size. DataLoader also deduplicates — if two posts have the same author, DataLoader fetches them once and reuses the result.

The trap: Not knowing DataLoader exists or explaining N+1 without the solution. In a senior interview, describing the problem without the fix is incomplete.

---

**Q (Medium): When would you choose gRPC-web over GraphQL for a browser client?**

Answer: gRPC-web is the better choice when: (1) Payload efficiency is critical — protobuf's binary encoding is 3–10x more compact than JSON and faster to serialize/deserialize, which matters for high-frequency data or very large payloads. (2) You need bidirectional or server-streaming that maps naturally to gRPC's streaming model. (3) The backend services are already gRPC and you want a typed contract directly, without a GraphQL translation layer. (4) The team has existing expertise in protobuf and gRPC tooling. GraphQL is better when: the browser team values schema flexibility and the ability to query arbitrary field combinations; the backend team doesn't want to manage a gRPC-web proxy (Envoy); or the API needs to be exposed to external consumers who don't use protobuf tooling. In practice, many architectures use gRPC service-to-service internally and expose a GraphQL or REST gateway to browser clients.

The trap: Claiming gRPC-web is the obvious performance choice without acknowledging the infrastructure overhead (proxy requirement) and tooling friction.

---

**Q (Low): What is an OpenAPI spec and how does it address REST's lack of built-in schema?**

Answer: OpenAPI (formerly Swagger) is a specification format for describing REST APIs — endpoints, request parameters, response shapes, authentication, and error codes — in a YAML or JSON document. It gives REST a machine-readable schema. From an OpenAPI spec, tools can: generate client SDKs in any language (TypeScript, Swift, Kotlin), generate server stubs, power interactive documentation (Swagger UI, Redoc), and validate request/response shapes in tests. The limitation: OpenAPI specs are separate artifacts that must be kept in sync with the actual implementation. Unlike GraphQL (where the schema is the implementation contract) or tRPC (where types flow from the code), OpenAPI specs can drift from the API's actual behavior. Runtime contract testing (Pact) or spec-first development (generate server stubs from the spec) mitigate this.

The trap: Thinking REST has no schema story. OpenAPI is the standard answer. Not knowing OpenAPI is a gap for a senior frontend engineer.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain over-fetching and under-fetching and how GraphQL addresses both
- [ ] Can explain why GraphQL is harder to cache at the HTTP layer and name two solutions
- [ ] Can explain what tRPC's type safety requires and why it's unsuitable for public APIs
- [ ] Can describe the N+1 problem and how DataLoader solves it
- [ ] Can name the right paradigm for: public API, full-stack TypeScript monorepo, microservice RPC
- [ ] Can explain what protobuf is and why gRPC payloads are smaller than REST/GraphQL

---
*Next: Real-time Communication — WebSockets, SSE, and Long Polling — when you need the server to push data to the browser without the client asking first.*
