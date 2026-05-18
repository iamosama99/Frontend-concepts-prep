# CORS Configuration & Preflight

## Quick Reference

| Header | Set by | Purpose |
|---|---|---|
| `Access-Control-Allow-Origin` | Server | Which origins may read the response |
| `Access-Control-Allow-Methods` | Server (preflight response) | Which methods are allowed |
| `Access-Control-Allow-Headers` | Server (preflight response) | Which request headers are allowed |
| `Access-Control-Allow-Credentials` | Server | Whether credentials (cookies) can be included |
| `Access-Control-Max-Age` | Server (preflight response) | Cache duration for preflight result |
| `Access-Control-Expose-Headers` | Server | Which response headers JS may read |
| `Origin` | Browser (automatic) | The requesting origin |

---

## What Is This?

CORS (Cross-Origin Resource Sharing) is a browser mechanism that controls whether JavaScript on one origin can read responses from a different origin. By default, the browser blocks cross-origin reads — this is the Same-Origin Policy (SOP). CORS is how servers opt in to allowing specific cross-origin access.

**Critical framing:** CORS does not prevent requests from being made. It controls whether the browser delivers the response to the requesting JavaScript. The request reaches the server either way (with the exception of preflighted requests — more below).

---

## What "Cross-Origin" Means

An origin is: scheme + host + port. Any difference makes it cross-origin:

```
https://api.example.com  ←→  https://app.example.com   CORS (different host)
https://example.com:443  ←→  https://example.com:8080   CORS (different port)
https://example.com      ←→  http://example.com          CORS (different scheme)
https://example.com/api  ←→  https://example.com/data    Same-origin (path differs, origin same)
```

> **Check yourself:** A browser makes a POST to an API on a different subdomain. Does CORS block the POST, or does the POST reach the server regardless?

---

## Simple Requests (No Preflight)

A request is "simple" if it meets all of these conditions:
- Method is `GET`, `HEAD`, or `POST`
- Content-Type is one of: `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`
- No custom headers beyond the CORS-safelisted headers (`Accept`, `Content-Language`, etc.)

Simple requests are sent directly — the browser sends the request, includes the `Origin` header automatically, and then checks the response's `Access-Control-Allow-Origin` header. If the origin isn't allowed, the browser blocks the JavaScript from reading the response.

```js
// Simple request — no preflight
fetch('https://api.example.com/data', {
  method: 'GET', // simple method
  // no custom headers
});
```

**The server must still respond correctly:**

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Content-Type: application/json

{"data": "..."}
```

---

## Preflighted Requests

Any request that isn't "simple" gets a preflight: an automatic `OPTIONS` request sent before the actual request. The browser asks the server: "Will you allow this kind of request from this origin?"

Triggers for preflight:
- Methods other than GET, HEAD, POST (e.g., `PUT`, `DELETE`, `PATCH`)
- `Content-Type: application/json` or any non-simple content type
- Custom headers (`Authorization`, `X-Custom-Header`, etc.)

```
Browser → OPTIONS https://api.example.com/resource
  Origin: https://app.example.com
  Access-Control-Request-Method: DELETE
  Access-Control-Request-Headers: Authorization, Content-Type

Server → 
  Access-Control-Allow-Origin: https://app.example.com
  Access-Control-Allow-Methods: GET, POST, DELETE
  Access-Control-Allow-Headers: Authorization, Content-Type
  Access-Control-Max-Age: 86400
  Status: 204

Browser (if preflight passed) → DELETE https://api.example.com/resource
  Origin: https://app.example.com
  Authorization: Bearer ...
  Content-Type: application/json
```

If the preflight fails (server doesn't return the right headers), the browser aborts — the actual request is **never sent**. This is the key difference from simple requests.

---

## Serving CORS Headers Correctly

### Single allowed origin

```http
Access-Control-Allow-Origin: https://app.example.com
```

### Dynamic origin validation (multiple allowed origins)

```js
// Node.js/Express
const ALLOWED_ORIGINS = new Set([
  'https://app.example.com',
  'https://admin.example.com',
]);

app.use((req, res, next) => {
  const origin = req.headers.origin;
  if (ALLOWED_ORIGINS.has(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Vary', 'Origin'); // tell caches this response varies by origin
  }
  next();
});
```

**`Vary: Origin` is required** when returning dynamic `Access-Control-Allow-Origin` values. Without it, a CDN or browser cache may serve a response with one origin's header to a different origin's request.

### Handling preflight

```js
app.options('*', (req, res) => {
  const origin = req.headers.origin;
  if (ALLOWED_ORIGINS.has(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, PATCH');
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    res.setHeader('Access-Control-Max-Age', '86400'); // cache preflight for 24 hours
  }
  res.status(204).end();
});
```

---

## Credentials (Cookies + Auth)

By default, cross-origin requests don't include cookies, Authorization headers, or TLS client certificates. To include them:

```js
// Client — must explicitly request credentials
fetch('https://api.example.com/profile', {
  credentials: 'include', // send cookies
});
```

```http
# Server — must explicitly allow credentials
Access-Control-Allow-Origin: https://app.example.com  ← must be specific, not *
Access-Control-Allow-Credentials: true
```

**`Access-Control-Allow-Origin: *` cannot be combined with `Access-Control-Allow-Credentials: true`.** The browser rejects this combination. The server must echo the specific requesting origin instead of using the wildcard.

---

## Common Misconfigurations

### 1. Reflecting any Origin blindly

```js
// DANGEROUS — allows any site to read your API responses with credentials
res.setHeader('Access-Control-Allow-Origin', req.headers.origin); // no validation
res.setHeader('Access-Control-Allow-Credentials', 'true');
```

This is equivalent to `Access-Control-Allow-Origin: *` but worse — it works with credentials. Any site can read authenticated API responses.

### 2. Wildcard with credentials

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

Browsers reject this (security constraint enforced client-side), but some servers still set it. Modern browsers display an error and block the response.

### 3. Not handling preflight for DELETE/PATCH

```js
// Server handles DELETE at /resource
// But only app.delete() is registered — app.options() returns 404
// Result: DELETE is never sent, appears as a CORS error to the client
```

Always handle `OPTIONS` for any endpoint that will receive non-simple requests.

### 4. Missing `Vary: Origin`

```http
# A CDN caches this response for user A (origin: app.example.com)
Access-Control-Allow-Origin: https://app.example.com

# User B (origin: attacker.com) gets the cached response with the wrong ACAO header
# Browser sees: response says app.example.com is allowed, but request is from attacker.com → error
# Or worse: CDN serves the wrong ACAO header, confusing the browser
```

### 5. Overly broad allowlists

```http
# Intended to allow subdomains — but regex escape error allows:
Access-Control-Allow-Origin: https://evilexample.com  ← if check was /example\.com/
```

Always use an exact allowlist, not a regex on the origin.

---

## CORS vs. JSONP

JSONP was the pre-CORS workaround: script tags aren't subject to SOP, so you'd inject `<script src="https://api.example.com/data?callback=fn">` and the server would respond with `fn({...})` — calling your JavaScript function. CORS replaced JSONP with a proper mechanism. JSONP is a security liability (executes arbitrary code from the server) and should never be used for new APIs.

---

## Interview Questions

**Q (High): Does CORS prevent a cross-origin request from reaching the server?**

Answer: For simple requests (GET, HEAD, POST with simple content types): no. The browser sends the request and only checks the `Access-Control-Allow-Origin` header in the response. If the origin isn't allowed, the browser blocks JavaScript from reading the response — but the request reached the server and may have caused side effects. This is why CORS doesn't protect against CSRF (the POST completes; the attacker just can't read the response). For preflighted requests (DELETE, PATCH, JSON content type, custom headers): yes — the browser sends an OPTIONS preflight first. If the preflight response doesn't grant permission, the actual request is never sent. This is the one case where CORS prevents the server from receiving the request.

**Q (High): What triggers a CORS preflight and what happens during one?**

Answer: A preflight is triggered when a request doesn't meet the "simple request" criteria: method is not GET/HEAD/POST, Content-Type is not a simple type (`application/json` triggers preflight), or custom headers like `Authorization` are included. The browser automatically sends an `OPTIONS` request to the same URL with `Access-Control-Request-Method` and `Access-Control-Request-Headers` indicating what the actual request will use. If the server responds with `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, and `Access-Control-Allow-Headers` headers that cover the actual request's requirements, the browser proceeds with the actual request. If the server returns 404 or missing CORS headers, the actual request is never made and JavaScript sees a network error. `Access-Control-Max-Age` caches the preflight result to avoid repeating it on subsequent requests.

**Q (High): Why can't `Access-Control-Allow-Origin: *` be used with cookies?**

Answer: The `*` wildcard means "any origin can read this response." Allowing any origin to read a response that includes cookies (via `Access-Control-Allow-Credentials: true`) would mean any site on the internet could make authenticated requests with the user's cookies and read the results. This would completely break any site's authentication — an attacker's page could read your email, banking data, or private profile by making credentialed cross-origin requests. Browsers enforce this: when `Access-Control-Allow-Credentials: true` is present, the `Access-Control-Allow-Origin` must be a specific origin. If it's `*`, the browser blocks the response regardless of credentials. The server must dynamically echo the specific requesting origin after validating it against an allowlist.

**Q (Medium): What is the `Vary: Origin` header and why is it required when dynamically setting CORS headers?**

Answer: When a server dynamically sets `Access-Control-Allow-Origin` to the requesting origin (different for different callers), the response content varies by the `Origin` request header. HTTP caches (CDNs, browser cache) use `Vary` to know which request headers determine whether a cached response can be reused. Without `Vary: Origin`, a cache might serve a response with `Access-Control-Allow-Origin: https://app.example.com` to a request from `https://admin.example.com`. The browser sees a mismatch — the ACAO header says app.example.com, but the request came from admin.example.com — and blocks the response. With `Vary: Origin`, the cache stores separate entries per origin value, serving each requester the correct response.

---

## Self-Assessment

- [ ] Explain what makes a request "simple" and how that determines whether a preflight fires
- [ ] Describe the preflight request/response cycle including which headers are required
- [ ] Explain why `Access-Control-Allow-Origin: *` with credentials is rejected by browsers
- [ ] Describe the security consequence of reflecting any origin without validation
- [ ] Explain why `Vary: Origin` is required for dynamic CORS origin headers

---
*Next: Auth & Authorization — JWT, OAuth2, OIDC, and HttpOnly cookie patterns.*
