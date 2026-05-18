# Trusted Types API

## Quick Reference

| Concept | Detail |
|---|---|
| Purpose | Browser-enforced prevention of DOM XSS by banning dangerous sink assignments from plain strings |
| Enforcement | Via CSP: `require-trusted-types-for 'script'` |
| Policy | Named object that vets strings and returns typed values (`TrustedHTML`, `TrustedScript`, `TrustedScriptURL`) |
| Sinks covered | `innerHTML`, `outerHTML`, `document.write`, `eval`, `setTimeout(string)`, `script.src`, etc. |
| Compatibility | Chrome 83+, Edge 83+; polyfill available for others |
| Relationship to CSP | Trusted Types is a CSP directive; works alongside `script-src` |

---

## What Is This?

Trusted Types is a browser API that eliminates DOM XSS at the platform level by enforcing that dangerous DOM sinks (`innerHTML`, `eval`, `script.src`, etc.) cannot accept plain strings — only typed objects produced by a vetted policy. If any code path tries to assign a plain string to `innerHTML`, the browser throws a `TypeError` before execution.

This turns DOM XSS from a runtime data-flow problem into a compile-time/policy-enforcement problem: you audit the policy functions once, and any future code that bypasses them is caught immediately.

---

## Why It Exists

DOM XSS requires data to flow from an attacker-controlled source (URL, postMessage, user input) to a dangerous sink (`innerHTML`, `eval`). Traditional approaches require auditing every call site — a combinatorial problem in large codebases. Trusted Types inverts the model: by making dangerous sinks reject plain strings, you audit only the policy functions that produce typed values. Any new code that tries to use a dangerous sink without going through the policy fails immediately with a clear error.

> **Check yourself:** If `innerHTML` can only accept a `TrustedHTML` object, and your policy is the only way to produce a `TrustedHTML` object, what does that mean for the audit surface?

---

## Enabling Trusted Types

Via CSP:

```http
Content-Security-Policy: require-trusted-types-for 'script'
```

With this header, any attempt to assign a plain string to a Trusted Types sink throws:

```js
document.body.innerHTML = '<b>Hello</b>';
// Uncaught TypeError: Failed to set the 'innerHTML' property on 'Element':
// This document requires 'TrustedHTML' assignment.
```

To also restrict which policies can be created (optional, stricter):

```http
Content-Security-Policy: require-trusted-types-for 'script'; trusted-types default myPolicy
```

This allows only policies named `default` and `myPolicy` to be created — any other `createPolicy()` call fails.

---

## Creating a Policy

```js
const policy = trustedTypes.createPolicy('myPolicy', {
  createHTML(input) {
    // This is the ONE place where you vet HTML strings
    // Use DOMPurify or similar — return a sanitized string
    return DOMPurify.sanitize(input);
  },

  createScript(input) {
    // Vet script strings — in practice, you almost never need this
    // Returning the input as-is is unsafe — only allow known-safe values
    throw new Error('createScript not allowed');
  },

  createScriptURL(input) {
    // Vet script URLs — ensure only allowed origins
    const url = new URL(input, location.origin);
    if (url.origin !== location.origin && url.origin !== 'https://trusted-cdn.example.com') {
      throw new Error('Script URL not allowed: ' + url.origin);
    }
    return input;
  },
});
```

### Using the policy

```js
// Instead of: element.innerHTML = someHtml
element.innerHTML = policy.createHTML(someHtml); // returns TrustedHTML object

// Instead of: scriptEl.src = someUrl
scriptEl.src = policy.createScriptURL(someUrl); // returns TrustedScriptURL object
```

The returned objects (`TrustedHTML`, `TrustedScript`, `TrustedScriptURL`) are accepted by their respective sinks. The browser only sees the type — it doesn't re-inspect the content.

---

## The `default` Policy

A policy named `'default'` is special — it intercepts string-to-sink assignments that weren't explicitly converted:

```js
trustedTypes.createPolicy('default', {
  createHTML(input) {
    console.warn('Unvetted innerHTML assignment:', input);
    return DOMPurify.sanitize(input); // still sanitize
  }
});
```

The `default` policy is a migration tool: it lets Trusted Types enforcement run without immediately breaking all existing `innerHTML = string` code. You see the violations in `createHTML` and progressively fix them. **It should not be the long-term state** — the goal is to have no calls relying on the default policy, meaning all sinks are explicitly vetted.

---

## Reporting Violations

Before enforcing, run in report-only mode to find all sink calls:

```http
Content-Security-Policy-Report-Only: require-trusted-types-for 'script'; report-uri /tt-report
```

This logs violations without blocking them. Use the reports to audit your codebase:

```json
{
  "csp-report": {
    "violated-directive": "require-trusted-types-for",
    "blocked-uri": "trusted-types-sink",
    "source-file": "/src/components/chat.js",
    "line-number": 42,
    "column-number": 23
  }
}
```

---

## Framework Behavior

Most modern frameworks are already Trusted Types compatible:

- **Angular** generates `TrustedHTML` for template rendering — fully compatible
- **React** doesn't use `innerHTML` for normal rendering; `dangerouslySetInnerHTML` is being updated to accept `TrustedHTML`
- **Vue** 3 is compatible
- **lit-html** generates `TrustedHTML`

The framework's own rendering path works without modification. The dangerous cases are:
- `dangerouslySetInnerHTML` with user-provided strings
- DOM manipulation in imperative code (jQuery, vanilla JS)
- Third-party library dependencies that use `innerHTML`

---

## Auditing Third-Party Dependencies

A Trusted Types violation from a third-party library means the library uses a dangerous sink. Options:

1. Use the `default` policy to intercept and sanitize their input
2. Find a Trusted Types-compatible version of the library
3. Wrap the library in a sandbox that runs before Trusted Types is enforced (pre-policy load)
4. Report the issue to the library maintainers

---

## What Trusted Types Does Not Protect Against

**Attribute-based injection:** `element.setAttribute('onclick', input)` is a dangerous sink for event handler injection, but not currently covered by Trusted Types (only `innerHTML`, `outerHTML`, `eval`, script src/text are covered).

**HTML reflection through templates:** If a server-side template reflects attacker input into HTML, Trusted Types on the client doesn't help — the script was already in the HTML before the client saw it. Trusted Types is specifically a DOM XSS protection (client-side injection from sources like the URL).

**JavaScript values:** Trusted Types doesn't type-check all JavaScript values — only the specific dangerous DOM sinks.

---

## Interview Questions

**Q (High): What problem does Trusted Types solve, and how does it change the security audit model?**

Answer: Trusted Types eliminates DOM XSS by banning plain strings from dangerous DOM sinks (`innerHTML`, `eval`, `script.src`, etc.). Without Trusted Types, preventing DOM XSS requires auditing every call site where user-controlled data touches a dangerous sink — a combinatorial problem in large codebases, and any new code that forgets the sanitization step is immediately a vulnerability. Trusted Types inverts this: all dangerous sinks reject plain strings, so any code that hasn't gone through an explicit policy fails immediately with a `TypeError`. The audit surface shrinks to just the policy functions — you audit them once, thoroughly, and any future code attempting to bypass them is caught at runtime. The number of policy functions is small and bounded; the number of potential call sites is not.

**Q (High): How does `require-trusted-types-for 'script'` in CSP enforce Trusted Types?**

Answer: When the browser loads a page with `require-trusted-types-for 'script'` in the CSP header, it activates Trusted Types enforcement for all JavaScript execution contexts on that page. Any attempt to assign a plain string to a Trusted Types sink — `element.innerHTML = "string"`, `eval("string")`, `scriptEl.src = "string"`, etc. — throws a `TypeError` synchronously, before the assignment takes effect. The browser requires that the value passed to the sink is a Trusted Types object (`TrustedHTML`, `TrustedScript`, `TrustedScriptURL`) produced by a registered policy. The `trusted-types` CSP directive (optionally paired with it) controls which policy names are allowed to be created — preventing attackers from just creating their own permissive policy.

**Q (Medium): What is the `default` Trusted Types policy and when is it appropriate?**

Answer: A policy named `'default'` is invoked automatically by the browser whenever a plain string assignment to a sink is attempted — any call site that hasn't been converted to use an explicit policy falls through to `default`. This makes it a migration tool: you can enable Trusted Types enforcement without immediately breaking all existing `innerHTML` code. The default policy intercepts the violations, you log them, and progressively fix each call site. The `default` policy is not the end goal — it still runs the input through whatever sanitization you put in `createHTML`, which is better than the status quo, but the long-term goal is zero reliance on `default`. Every call site should have its own explicit policy use, or be refactored to use safe alternatives (`textContent` instead of `innerHTML` for plain text).

**Q (Low): Why doesn't Trusted Types protect against server-side reflected XSS?**

Answer: Trusted Types is a DOM XSS protection — it operates on the client, governing what JavaScript can do to the DOM at runtime. Reflected XSS sends malicious HTML/script from the server as part of the HTML response, before any client-side code runs. The browser receives and parses the HTML, executing any inline scripts, before Trusted Types has any opportunity to intercept. Trusted Types can't intercept the browser's initial HTML parsing. It only intercepts JavaScript operations — calls to `innerHTML`, `eval`, etc. — made after the page has loaded. Server-side output encoding and CSP's `script-src` with nonces/hashes are the defenses against reflected and stored XSS.

---

## Self-Assessment

- [ ] Explain what the `require-trusted-types-for 'script'` CSP directive does
- [ ] Write a policy that sanitizes HTML using DOMPurify and allows only trusted script URLs
- [ ] Describe why the audit surface shrinks with Trusted Types compared to auditing every `innerHTML` call
- [ ] Explain the `default` policy's purpose and why it's a migration tool, not a final state
- [ ] Describe why Trusted Types doesn't protect against server-side reflected XSS

---
*Next: Phase 14 — TypeScript & Type System Architecture — structural typing, generics, conditional types, module augmentation, and more. Code examples switch to TypeScript from here.*
