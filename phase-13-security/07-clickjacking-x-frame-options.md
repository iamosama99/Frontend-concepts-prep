# Clickjacking: X-Frame-Options & frame-ancestors

## Quick Reference

| Header | Values | Scope |
|---|---|---|
| `X-Frame-Options: DENY` | Never frame this page | All origins including self |
| `X-Frame-Options: SAMEORIGIN` | Only same-origin frames allowed | Same origin only |
| `X-Frame-Options: ALLOW-FROM uri` | Specific origin (deprecated, poor browser support) | One origin |
| CSP `frame-ancestors 'none'` | Never frame this page | More flexible, supersedes X-Frame-Options |
| CSP `frame-ancestors 'self'` | Same-origin frames only | |
| CSP `frame-ancestors https://trusted.com` | Specific origins | Supports multiple origins |

**Recommendation:** Use CSP `frame-ancestors` as the primary control. Add `X-Frame-Options: DENY` as a fallback for older browsers.

---

## What Is This?

Clickjacking is an attack where an attacker loads your page in a transparent iframe overlaid on their own page. The victim believes they're clicking on the attacker's content but is actually clicking on your page — submitting forms, confirming purchases, changing settings, or performing other actions on your authenticated application.

```
Victim sees:  "Click here to win a prize!"
              [BUTTON]

What's really happening:
  Attacker's page:
    <iframe src="https://bank.com/confirm-transfer?to=attacker" style="opacity:0; position:absolute; top: 0; left: 0;" />
    <button style="position:absolute; top:same; left:same">Click here to win a prize!</button>

  The victim's click hits the invisible iframe — confirming the transfer.
```

---

## Why Invisible iframes Work

iframes inherit the victim's session — cookies are sent automatically with the iframe's request to the framed site. The victim is authenticated. The attacker's page positions the iframe so a button or link aligns with a meaningful action on the victim's site.

> **Check yourself:** How does SameSite=Strict on the session cookie affect clickjacking attacks?

---

## X-Frame-Options

The legacy mechanism. A response header that tells the browser whether this page may be embedded in a frame.

```http
X-Frame-Options: DENY
```

`DENY` — the page cannot be displayed in any frame, regardless of origin.

```http
X-Frame-Options: SAMEORIGIN
```

`SAMEORIGIN` — the page can be framed only by pages on the same origin.

```http
X-Frame-Options: ALLOW-FROM https://partner.example.com
```

**Deprecated and unreliably supported.** Specifies one allowed origin. Not supported in Chrome. Use CSP `frame-ancestors` instead.

**Limitation of X-Frame-Options:** Only one value, only one origin, no support for `ALLOW-FROM` in modern browsers. CSP `frame-ancestors` replaced it for all multi-origin and nuanced framing rules.

---

## CSP `frame-ancestors`

The modern, flexible control. Part of the Content-Security-Policy header:

```http
Content-Security-Policy: frame-ancestors 'none'
```

Completely prevents the page from being framed — same as `X-Frame-Options: DENY`.

```http
Content-Security-Policy: frame-ancestors 'self'
```

Allow framing only by same-origin pages.

```http
Content-Security-Policy: frame-ancestors 'self' https://partner.example.com https://app.example.com
```

Allow multiple specific origins — something `X-Frame-Options` can't do.

**CSP `frame-ancestors` takes precedence over `X-Frame-Options`** in browsers that support both (all modern browsers). Send both headers for backward compatibility with legacy browsers:

```http
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'
```

---

## When Framing Is Legitimate

Some applications need to allow framing:
- **Embeddable widgets** (chat widgets, payment buttons, maps)
- **Portals** (enterprise dashboards that iframe sub-applications)
- **Cross-domain communication via postMessage**

For these cases, explicitly allowlist the trusted parent origins:

```http
Content-Security-Policy: frame-ancestors https://dashboard.company.com https://portal.company.com
```

For embeddable widgets that need to run in arbitrary third-party contexts, the widget itself must be designed for framing. The parent page cannot be protected — the widget page's frame-ancestors must allow it.

---

## SameSite Cookies as a Complementary Defense

`SameSite=Strict` on the session cookie prevents the cookie from being sent when the page is loaded inside an iframe on a cross-site origin:

```http
Set-Cookie: session=abc; SameSite=Strict; Secure; HttpOnly
```

If the session cookie isn't sent, the iframed page loads the unauthenticated view — the attacker sees a login page, not the authenticated account. This makes clickjacking attacks ineffective for sites using `SameSite=Strict`.

`SameSite=Lax` also helps — it doesn't send cookies for subresource requests like iframes. Only `SameSite=None` allows cookies in cross-site iframes.

**Defense in depth:** Use `frame-ancestors` + SameSite cookies + CSRF tokens. Each layer stops different attack variants.

---

## Frame-Busting JavaScript (The Old Way — Don't Use)

Before `X-Frame-Options` and CSP, developers used JavaScript to detect and break out of iframes:

```js
// Old frame-busting — fragile
if (top !== self) {
  top.location = self.location; // try to redirect parent to this page
}
```

**This doesn't work.** Attackers defeat frame-busting by:
- Using `sandbox` attribute on the iframe (disables JavaScript)
- Setting `X-Frame-Options` is more reliable on the server side
- Using `onbeforeunload` to cancel the redirect

Frame-busting JS is security theater. Use server-side headers.

---

## Gotchas

**`frame-ancestors` is not inherited.** Each page that you don't want framed must set its own header. If you set it on `/`, pages at `/settings` don't automatically inherit it unless you configure your server to send it for all responses.

**`frame-ancestors` is not respected by `<meta>` CSP.** `frame-ancestors` (and `report-uri`, `sandbox`) cannot be set via `<meta http-equiv="Content-Security-Policy">` — they only work as HTTP response headers. This is intentional: if a meta tag could prevent framing, the attacker could inject a `<meta>` tag that removes the restriction.

**`X-Frame-Options` doesn't support wildcards or multiple origins.** `ALLOW-FROM` only accepts one URI and isn't supported by Chrome. For any multi-origin framing allowlist, CSP is required.

---

## Interview Questions

**Q (High): What is clickjacking and how does X-Frame-Options/CSP frame-ancestors prevent it?**

Answer: Clickjacking overlays a transparent iframe of a victim's authenticated page on top of an attacker's page, aligned so the victim's clicks on the attacker's UI actually trigger actions on the victim's page. The victim's session cookie is sent with the iframe request (if SameSite allows it), so the framed page is fully authenticated. `X-Frame-Options: DENY` or CSP `frame-ancestors 'none'` instructs the browser not to render the page inside any iframe. The browser checks the response headers when loading the iframe target — if the header disallows framing, the iframe renders blank and no clickjacking is possible. Modern browsers enforce this reliably, making frame-busting JavaScript obsolete.

**Q (High): What is the difference between `X-Frame-Options` and CSP `frame-ancestors`?**

Answer: `X-Frame-Options` supports only two useful values: `DENY` and `SAMEORIGIN`. `ALLOW-FROM` for a specific origin is deprecated and not supported in Chrome. `frame-ancestors` in CSP is the modern replacement: it supports multiple origins in one directive, supports wildcards for subdomains, is more precisely defined, and takes precedence over `X-Frame-Options` in modern browsers. The practical recommendation is to use CSP `frame-ancestors` as the primary control and add `X-Frame-Options: DENY` as a fallback for legacy browsers. One specific difference: `frame-ancestors` cannot be set via a `<meta>` CSP tag — only as an HTTP header. This is intentional security: a server-controlled header can't be overridden by injected HTML.

**Q (Medium): How does SameSite=Strict on session cookies complement frame-ancestors protection for clickjacking?**

Answer: `frame-ancestors 'none'` prevents the page from being framed. `SameSite=Strict` provides a second layer: even if `frame-ancestors` is misconfigured or missing, the session cookie isn't sent when the page is loaded in a cross-site iframe. Without the session cookie, the framed page shows the unauthenticated view — the login form — rather than the victim's account. The attacker's clickjack overlay is targeting an empty/login page, not the victim's authenticated session. This makes the attack fail even without `frame-ancestors`. Defense in depth: `frame-ancestors` blocks framing; SameSite=Strict prevents authenticated sessions from working inside cross-site frames.

**Q (Low): Why did frame-busting JavaScript fail and what replaces it?**

Answer: Frame-busting JS (`if (top !== self) top.location = self.location`) is easily defeated. The attacker can add `sandbox` to the iframe — sandboxed iframes block JavaScript execution (`allow-scripts` can be added back, but specifically without `allow-top-navigation`). The attacker can also catch the `onbeforeunload` event on the parent page to cancel the redirect. Browser quirks around frame-busting added more bypasses. The fundamental problem is that frame-busting relies on JavaScript executing inside a context the attacker controls. The correct solution is a server-sent HTTP header — the browser enforces it before rendering, before any JavaScript runs, and the attacker has no mechanism to override a correctly sent response header from the target server.

---

## Self-Assessment

- [ ] Describe the clickjacking attack vector and why invisible iframes are dangerous
- [ ] Explain the difference between `X-Frame-Options: DENY` and CSP `frame-ancestors 'none'`
- [ ] Write the headers for a page that should only be frameable by `dashboard.company.com`
- [ ] Explain why `frame-ancestors` cannot be set via a meta CSP tag
- [ ] Describe how SameSite=Strict provides a second layer of clickjacking protection

---
*Next: Trusted Types API — browser-enforced DOM XSS prevention using policy objects.*
