# Content Security Policy (CSP): Nonces, Hashes, strict-dynamic

## Quick Reference

| Directive | Controls |
|---|---|
| `default-src` | Fallback for all resource types |
| `script-src` | JavaScript sources |
| `style-src` | CSS sources |
| `connect-src` | fetch, XHR, WebSocket, EventSource |
| `img-src` | Images |
| `font-src` | Web fonts |
| `frame-src` | Iframes |
| `report-uri` / `report-to` | Where to send violation reports |

| Keyword | Meaning |
|---|---|
| `'none'` | Block everything |
| `'self'` | Same origin only |
| `'unsafe-inline'` | Allow inline scripts/styles (defeats XSS protection) |
| `'unsafe-eval'` | Allow eval, Function(), setTimeout(string) |
| `'nonce-<base64>'` | Allow specific inline script/style with matching nonce |
| `'sha256-<hash>'` | Allow specific inline script/style by hash |
| `'strict-dynamic'` | Propagate trust from nonce/hash scripts to their dynamically loaded scripts |

---

## What Is This?

Content Security Policy is an HTTP response header (or `<meta>` tag) that tells the browser which resources are allowed to load and execute on a page. It's a defense-in-depth layer that limits the damage from XSS attacks: even if an attacker injects a script, CSP prevents that script from executing if its source or inline content isn't explicitly allowed.

```http
Content-Security-Policy: script-src 'self' 'nonce-abc123'; object-src 'none'; base-uri 'none'
```

---

## Why Default CSP Approaches Fail

### The `unsafe-inline` trap

The simplest script policy seems to be:

```http
Content-Security-Policy: script-src 'self' 'unsafe-inline'
```

This allows any inline `<script>` tag to execute. But that's exactly what XSS injects. Allowing `'unsafe-inline'` completely defeats CSP's XSS protection — it's as if CSP didn't exist for script injection.

### Allowlisting domains is weak

```http
Content-Security-Policy: script-src 'self' https://cdn.example.com
```

If any file on `cdn.example.com` is attacker-controlled (a file upload endpoint, a public JSONP endpoint), the attacker can serve their script from the allowed domain. Domain allowlisting is better than nothing but fragile.

> **Check yourself:** Why does adding `'unsafe-inline'` to `script-src` defeat CSP's XSS protection entirely?

---

## The Nonce-Based Policy

A nonce (number used once) is a random, per-request, base64-encoded value the server generates and puts both in the CSP header and on allowed `<script>` tags:

```python
# Server generates a random nonce per request
import secrets, base64
nonce = base64.b64encode(secrets.token_bytes(16)).decode()

# Set the header:
response.headers['Content-Security-Policy'] = (
    f"script-src 'nonce-{nonce}' 'strict-dynamic'; object-src 'none'"
)

# Inject nonce into HTML template:
# <script nonce="{{ nonce }}">...</script>
```

```html
<!-- These scripts have the nonce — they execute -->
<script nonce="abc123">initApp();</script>
<script nonce="abc123" src="/app.js"></script>

<!-- This injected script has no nonce — blocked -->
<script>stealCookie();</script>
```

The attacker doesn't know the nonce value (it's generated fresh per request), so injected scripts can't include it, and CSP blocks them.

**Nonce requirements:**
- Minimum 128 bits of entropy (16 bytes from CSPRNG)
- Regenerated on every response — same nonce used twice becomes predictable
- Transmitted only via HTTP header (not in URLs, cookies, or HTML attributes other than nonce)
- Must be generated server-side in SSR — static files can't use nonces without a server wrapper

---

## Hash-Based Policy

For static inline scripts where the content never changes, a hash of the script content can be used instead of a nonce:

```http
Content-Security-Policy: script-src 'sha256-abc123...=='
```

```html
<!-- Script with exact content that matches the hash -->
<script>const x = 1;</script>
```

The hash is computed over the exact byte content of the `<script>` tag's content (not including the tags themselves). Any change — even whitespace — invalidates the hash.

**When to use hashes:**
- Static pages with fixed inline scripts (e.g., analytics initialization snippets)
- When server-side templating isn't available
- CSS `<style>` blocks that are static

**Limitation:** Can't be used for dynamically generated inline scripts (different content per request). Use nonces for those.

---

## `'strict-dynamic'`

The nonce/hash approach has a problem: modern bundlers load JavaScript dynamically. A React or Webpack app adds `<script>` tags at runtime to lazy-load chunks. These dynamically added scripts don't have the nonce.

`'strict-dynamic'` solves this: **scripts that run with nonce/hash trust are allowed to load additional scripts dynamically**, even without a nonce on those additional scripts.

```http
Content-Security-Policy: script-src 'nonce-abc123' 'strict-dynamic'; object-src 'none'
```

```js
// This inline script has the nonce — it executes
// It creates a <script> tag — strict-dynamic allows this
const script = document.createElement('script');
script.src = '/chunk.js'; // no nonce needed — trust propagated from parent
document.head.appendChild(script);
```

`'strict-dynamic'` also ignores host allowlists (`'self'`, domain entries) — only nonce/hash grants script trust. This is actually stronger: it eliminates the "find a script on an allowed domain" bypass.

**Complete modern policy:**

```http
Content-Security-Policy: 
  script-src 'nonce-{random}' 'strict-dynamic';
  object-src 'none';
  base-uri 'none'
```

- `nonce-{random}` — trust scripts with the right nonce
- `'strict-dynamic'` — propagate trust to dynamically loaded scripts
- `object-src 'none'` — block Flash and other plugins (dead but still blocked)
- `base-uri 'none'` — prevent `<base>` tag injection that redirects relative URLs

---

## CSP Reporting

CSP violations generate reports. `report-uri` (deprecated) or `report-to` sends these to a backend:

```http
Content-Security-Policy: script-src 'self'; report-uri /csp-report
# or modern:
Content-Security-Policy: script-src 'self'; report-to csp-endpoint
Report-To: {"group":"csp-endpoint","max_age":86400,"endpoints":[{"url":"/csp-report"}]}
```

Report-only mode — observe violations without blocking:

```http
Content-Security-Policy-Report-Only: script-src 'self' 'nonce-{nonce}'; report-uri /csp-report
```

Use report-only mode to test a policy before enforcing it. Monitor reports for legitimate violations (from browser extensions, third-party scripts) before switching to enforcement mode.

---

## CSP and Common Challenges

### Third-party scripts

Analytics, chat widgets, and A/B testing scripts often require `'unsafe-inline'` or specific domain entries. Options:

1. Load the script with a nonce if you control when it's added to HTML
2. Use a facade/loader: a nonce'd script fetches and injects the third-party script (strict-dynamic propagates trust)
3. Accept the specific domain allowlist entry for that script
4. Use a Tag Manager behind a nonce (GTM can be loaded via nonce, then manages third-party scripts)

### Inline event handlers

`onclick="..."` and other inline event handlers are blocked by CSP when `'unsafe-inline'` is absent. Fix: move all event handling to JavaScript files.

### `eval` and `new Function`

Many template engines, i18n libraries, and some animation libraries use `eval`. These require `'unsafe-eval'`. Alternatives: use CSP-compatible libraries, pre-compile templates at build time.

---

## Meta CSP

Can be delivered via `<meta>` tag for static sites without server control:

```html
<meta http-equiv="Content-Security-Policy" content="script-src 'nonce-abc' 'strict-dynamic'">
```

Limitation: meta CSP doesn't support `frame-ancestors`, `report-uri`, or `sandbox` directives. Also, the nonce in the meta tag must be hardcoded — meta CSP can't use per-request nonces unless the HTML is dynamically generated.

---

## Interview Questions

**Q (High): What is a CSP nonce, why must it be generated per-request, and what does it protect against?**

Answer: A nonce is a random, single-use token generated server-side and embedded in both the `Content-Security-Policy` header (`nonce-<value>`) and the `nonce` attribute of allowed `<script>` and `<style>` tags. The browser only executes scripts whose nonce attribute matches the value in the CSP header. An XSS attacker who injects a `<script>` tag into the page cannot include the correct nonce — they don't know it. The server generates it fresh for every response using a CSPRNG with at least 128 bits of entropy, so it's unpredictable. If the nonce were static (same value on every page load), an attacker who observed it once (via browser history, server logs, Referer headers) could reuse it. Per-request generation ensures each nonce is useless after the response is served.

**Q (High): What does `'strict-dynamic'` do and why is it needed for modern single-page applications?**

Answer: Modern SPAs and bundlers routinely add `<script>` tags dynamically at runtime — for code splitting, lazy routes, and async imports. These dynamically inserted scripts don't have a nonce (they're added by JavaScript, not by the server's HTML template). Without `'strict-dynamic'`, CSP blocks them all, breaking the application. `'strict-dynamic'` says: any script that executes with the trust of a nonce or hash is allowed to dynamically insert other scripts, and those scripts are also trusted. Trust propagates from the nonce'd entry points to everything they load. Additionally, `'strict-dynamic'` deliberately ignores host allowlist entries (`'self'`, specific domains) — only nonce/hash matters. This is actually a security improvement: it eliminates the "find a JSONP endpoint on an allowed domain" bypass technique.

**Q (Medium): Why is `'unsafe-inline'` in `script-src` the same as having no CSP at all for XSS protection?**

Answer: The entire purpose of CSP's XSS protection is to block injected inline scripts — the output of a successful XSS attack. XSS works by causing the browser to interpret attacker-controlled strings as executable HTML, resulting in inline `<script>` tags or inline event handlers. `'unsafe-inline'` tells the browser to execute all inline scripts unconditionally, which is exactly what an attacker needs. A CSP with `'unsafe-inline'` will block scripts from unexpected external domains (providing some protection against loading external malware), but allows all injected inline scripts — meaning it provides zero protection against the most common XSS payload. The correct approach is: nonces for dynamically generated inline scripts, hashes for fixed inline scripts, and `'strict-dynamic'` for dynamically loaded scripts.

**Q (Low): A page has a valid strict nonce-based CSP, but a user's browser extension injects a script. Will the violation report fire?**

Answer: Yes. The browser extension's injected script doesn't have the correct nonce, so CSP fires a violation and sends a report to the `report-uri` or `report-to` endpoint. This is a significant source of false positives in CSP violation dashboards — browser extensions, password managers, and screen readers legitimately inject scripts that violate CSP. When operating in report-only mode to tune a new policy, filter out extension-sourced violations before concluding that a policy is ready for enforcement. Common indicators in reports: the violating script URL contains the extension's identifier (chrome-extension://, moz-extension://) or the blocked URI is 'inline' in a way that maps to a known extension pattern.

---

## Self-Assessment

- [ ] Explain why `'unsafe-inline'` defeats CSP's XSS protection
- [ ] Describe the nonce lifecycle: generation, embedding in HTML, embedding in header, browser validation
- [ ] Explain what `'strict-dynamic'` does and why SPAs need it
- [ ] Write a complete strict CSP header for a server-rendered app with dynamic imports
- [ ] Describe report-only mode and when to use it

---
*Next: CORS Configuration & Preflight — simple vs. preflighted requests, credentials, and common misconfigurations.*
