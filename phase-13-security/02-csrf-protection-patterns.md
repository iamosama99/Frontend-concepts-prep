# Cross-Site Request Forgery (CSRF) Protection Patterns

## Quick Reference

| Protection | How it works | Best for |
|---|---|---|
| `SameSite=Strict` | Cookie not sent on cross-site requests | Session cookies for non-linked sites |
| `SameSite=Lax` | Cookie sent only on top-level GET navigations | Most session cookies (modern default) |
| `SameSite=None; Secure` | Cookie always sent (opt-in cross-site) | Third-party cookies, requires Secure |
| CSRF token (Synchronizer Token) | Server issues a secret; form submits it back | Traditional server-rendered apps |
| Double Submit Cookie | Cookie value echoed in request header/body | Stateless, distributed backends |
| Custom request header | Non-standard header breaks CORS preflight for cross-origin requests | JSON APIs, SPAs |

---

## What Is This?

CSRF (Cross-Site Request Forgery) is an attack where a malicious site tricks a victim's browser into making a request to a different site where the victim is authenticated. The browser automatically includes the victim's cookies on that request — so the target site can't distinguish it from a legitimate request.

```
Victim is logged into bank.com.
Attacker's site sends: POST https://bank.com/transfer?to=attacker&amount=5000
The victim's browser includes the bank.com session cookie automatically.
The bank processes the transfer as if the victim requested it.
```

CSRF is fundamentally a browser behavior problem — cookies are sent automatically with every request to the matching origin, regardless of where the request originated.

> **Check yourself:** CSRF works because cookies are sent automatically. Why doesn't XSS have the same automatic restriction?

---

## SameSite Cookie Attribute

The most important modern CSRF defense. Controls when the browser sends a cookie with cross-site requests.

```http
Set-Cookie: session=abc123; SameSite=Lax; Secure; HttpOnly
```

### `SameSite=Strict`

Cookie is **never** sent on cross-site requests — not on form submissions, link navigations, or resource loads.

```
User on evil.com clicks a link to bank.com → bank.com's session cookie NOT sent
User on evil.com POSTs a form to bank.com → bank.com's session cookie NOT sent
User navigates directly to bank.com → cookie sent (same-site)
```

Strongest protection. Downside: if a user clicks a link to your site from an email, they arrive logged out — the session cookie wasn't sent. Jarring for sites where cross-site navigation should feel seamless.

### `SameSite=Lax` (modern default)

Cookie is **sent on top-level GET navigations** (clicking a link) but not on subresource requests, cross-site POSTs, or iframes.

```
User on evil.com clicks a link to bank.com → cookie IS sent (top-level GET)
User on evil.com submits a form POST to bank.com → cookie NOT sent
evil.com's <img src="bank.com/api"> → cookie NOT sent
evil.com's iframe of bank.com → cookie NOT sent
```

Since 2021, Chrome (and most browsers) default to `Lax` for cookies without a `SameSite` attribute. This makes most CSRF attacks fail by default even without explicit server configuration.

### `SameSite=None; Secure`

Cookie sent on all cross-site requests. Used for third-party cookies (analytics, payment widgets, embedded content). Requires `Secure` (HTTPS-only) or the browser rejects it.

---

## The State-Changing GET Problem

`SameSite=Lax` allows cross-site GET. If your application has state-changing GET endpoints (e.g., `/logout`, `/confirm?token=X`), CSRF is still possible:

```html
<!-- evil.com — causes victim to log out of target.com -->
<img src="https://target.com/logout" />
```

**Fix:** Never use GET for state-changing operations. GET must be safe and idempotent (REST principle, also a security requirement).

---

## CSRF Tokens (Synchronizer Token Pattern)

For server-rendered forms or APIs that can't rely solely on SameSite:

```
1. Server generates a random, unguessable token and stores it in the session
2. Token is embedded in every HTML form or returned via API
3. On form submission or API call, client sends the token in the request body or header
4. Server validates the token matches the session — if not, reject
```

```html
<!-- Server-rendered form -->
<form method="POST" action="/transfer">
  <input type="hidden" name="csrf_token" value="{{session.csrf_token}}" />
  <input type="text" name="amount" />
  <button type="submit">Transfer</button>
</form>
```

**Why it works:** A cross-site forger can't read the CSRF token from your page (same-origin policy blocks cross-origin reads). They can trigger a request but can't include the correct token. The server rejects the request.

**Token requirements:**
- Cryptographically random (use `crypto.randomBytes` or `secrets.token_hex`)
- Tied to the user session (not global)
- Single-use or rotated periodically
- Validated server-side on every state-changing request

---

## Double Submit Cookie Pattern

Stateless alternative when server sessions aren't available (distributed backends):

```
1. Server sets a random cookie: Set-Cookie: csrf=RANDOM_VALUE; SameSite=None; Secure
2. Client reads the cookie value (must be non-HttpOnly to be readable by JS)
3. Client sends the value as a request header: X-CSRF-Token: RANDOM_VALUE
4. Server verifies the header value matches the cookie value
```

```js
// Client reads the csrf cookie and echoes it in the header
const csrfToken = getCookie('csrf');

fetch('/api/transfer', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-CSRF-Token': csrfToken, // custom header
  },
  body: JSON.stringify({ amount: 100 }),
});
```

**Why it works:** A cross-site request can send cookies automatically (that's the whole CSRF problem) but cannot add custom headers to cross-site requests — CORS preflight blocks that. A forger can trigger the cookie being sent but cannot add the `X-CSRF-Token` header matching the cookie value from a different origin.

**Important:** The double-submit cookie must be different from the session cookie and must be readable by JavaScript (not HttpOnly).

---

## Custom Request Headers (for JSON APIs / SPAs)

The simplest CSRF protection for APIs that only accept JSON:

```js
// Client always sends Content-Type: application/json
fetch('/api/transfer', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(data),
});
```

HTML forms cannot send `Content-Type: application/json`. A CSRF form POST always uses `application/x-www-form-urlencoded` or `multipart/form-data`. A cross-site request with a custom content type triggers a CORS preflight — which the browser sends before the actual request, and which the server can (and should) reject.

**Server validation:**

```js
// Express middleware — reject if Content-Type is not JSON for API routes
app.use('/api', (req, res, next) => {
  if (req.method !== 'GET' && !req.is('application/json')) {
    return res.status(415).json({ error: 'JSON required' });
  }
  next();
});
```

This works because non-simple custom headers (including `Content-Type: application/json`) require CORS preflight from cross-origin requests. The server's CORS configuration controls whether the preflight succeeds.

---

## Are SPAs Safe from CSRF by Default?

A fully client-rendered SPA that:
1. Uses `fetch` with `Content-Type: application/json` (or other non-simple content types)
2. Stores tokens in memory (not cookies) and sends them as `Authorization: Bearer` headers

...is naturally immune to CSRF because:
- Custom headers cannot be set by cross-site HTML forms
- `Authorization: Bearer` tokens are never sent automatically by the browser (unlike cookies)

However, if the SPA uses cookies for authentication (common for HttpOnly cookies to prevent XSS), CSRF is still relevant and SameSite + CSRF tokens are needed.

---

## CSRF vs. CORS

These are commonly confused.

**CORS** controls which origins can read cross-origin responses. A cross-origin request can still *reach* the server and cause side effects — CORS just determines whether the browser gives the JavaScript response data.

**CSRF** is about making unauthorized state-changing requests. A CSRF attack succeeds the moment the server processes the request — the attacker doesn't need to read the response.

CORS doesn't protect against CSRF. A server with `Access-Control-Allow-Origin: *` still processes every cross-origin POST — it just serves the response to any origin. The damage is done whether or not the attacker reads the response.

---

## Interview Questions

**Q (High): What is CSRF and why does SameSite=Lax protect against it in most cases?**

Answer: CSRF exploits the browser's automatic cookie behavior: cookies for a domain are sent with every request to that domain, regardless of which site initiated the request. An attacker's page can trigger a request to bank.com and the victim's session cookie goes along automatically. `SameSite=Lax` prevents the session cookie from being sent on cross-site requests *except* top-level GET navigations. Since CSRF attacks that cause harm require POST (or other state-changing methods), blocking cross-site cookies on POST requests eliminates most attacks. `Lax` allows cross-site GET navigation (clicking a link) so users clicking your links from other sites arrive logged in — which is the desired UX. It blocks cross-site form posts, iframe requests, and subresource loads — the attack vectors.

**Q (High): Explain the double-submit cookie pattern. Why can a cross-site attacker not defeat it?**

Answer: The double-submit cookie stores a random value in a cookie and requires the client to echo that same value in a request header or body. The server validates they match. An attacker initiating a cross-site request has the victim's cookies sent automatically (that's the CSRF mechanism), but they cannot read or modify the cookie value from their origin — the same-origin policy blocks cross-origin cookie reading. They also cannot add the matching custom header to a cross-site request — CORS preflight would fire and the server would reject it. The attacker can cause the cookie to be sent but can't provide the matching header value because they can't read the cookie to copy it. Limitation: the CSRF cookie must not be HttpOnly (JS must read it), which creates a slight XSS exposure for that specific cookie.

**Q (Medium): Why doesn't CORS protect against CSRF?**

Answer: CORS controls whether the browser allows JavaScript to read a cross-origin response — it's a client-side protection on response access. CSRF attacks don't need to read the response; they only need the request to be processed. A CSRF-exploitable endpoint processes the incoming POST, executes the state change, and returns a response. Whether the browser delivers that response to the attacker's JavaScript is irrelevant — the damage (fund transfer, password change, data deletion) is already done. Even with `Access-Control-Allow-Origin: *`, the server still receives and processes every cross-origin request. CORS and CSRF solve orthogonal problems: CORS protects response confidentiality; CSRF tokens/SameSite protect request authorization.

**Q (Low): A pure JSON SPA uses `fetch` with `Content-Type: application/json` and stores the auth token in memory as `Authorization: Bearer`. Is it vulnerable to CSRF?**

Answer: No. Two independent protections make it immune. First, the `Authorization: Bearer` header is never sent automatically — the browser only auto-sends cookies, not custom auth headers. A CSRF attacker can trigger a cross-site request but cannot add the Authorization header from a different origin. Second, `Content-Type: application/json` is a non-simple content type, so any cross-site request with this header triggers a CORS preflight. The server's CORS configuration rejects preflights from untrusted origins, so the actual request never reaches the server. Both mechanisms independently block CSRF. The risk returns if the SPA switches to cookie-based auth (HttpOnly session cookie), at which point SameSite + CSRF tokens are needed.

---

## Self-Assessment

- [ ] Explain what CSRF is and why cookies enable it
- [ ] Describe the difference between SameSite=Strict and SameSite=Lax and when each is appropriate
- [ ] Explain the synchronizer token pattern — how the token is generated, transmitted, and validated
- [ ] Describe the double-submit cookie pattern and why a cross-origin attacker can't defeat it
- [ ] Explain why a JSON API with Bearer tokens is inherently CSRF-safe

---
*Next: Content Security Policy — nonces, hashes, strict-dynamic, and blocking inline scripts.*
