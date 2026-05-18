# Subresource Integrity (SRI)

## Quick Reference

| Concept | Detail |
|---|---|
| Purpose | Verify that a CDN-served file hasn't been tampered with |
| Mechanism | Hash of expected content embedded in HTML; browser computes hash of received content and compares |
| Algorithms | `sha256`, `sha384`, `sha512` (sha384 is the most common practical choice) |
| Syntax | `integrity="sha384-<base64hash>"` on `<script>` and `<link>` |
| CORS requirement | Cross-origin SRI requires the server to send `Access-Control-Allow-Origin` |
| Limitation | Only protects against tampering; doesn't protect against the CDN serving the wrong version |

---

## What Is This?

Subresource Integrity (SRI) is a browser security feature that lets you cryptographically verify that files fetched from external sources (CDNs, third-party hosts) haven't been modified. You embed a hash of the expected file content in your HTML. When the browser fetches the file, it computes the same hash and checks it matches. If the CDN serves a modified or compromised file, the hash won't match and the browser refuses to execute it.

---

## Why It Exists

Loading scripts from CDNs is common — jQuery, React, chart libraries. If the CDN is compromised (supply chain attack), the attacker can modify the served files to inject malicious code into every site that loads from that CDN. SRI ensures your page only runs files that exactly match what you audited when you wrote the HTML.

```
Without SRI:
  CDN compromised → attacker modifies jquery.min.js → every site loads malware

With SRI:
  CDN compromised → attacker modifies jquery.min.js → hash doesn't match → browser blocks it
```

> **Check yourself:** If you host a file on your own server (not a CDN), do you need SRI for it?

---

## Usage

```html
<!-- Script with SRI -->
<script
  src="https://cdn.jsdelivr.net/npm/jquery@3.7.1/dist/jquery.min.js"
  integrity="sha384-1H217gwSVyLSIfaLxHbE7dRb3v4mYCKbpQvzx0cegeju1MVsGrX5xXxAvs/HgeFs"
  crossorigin="anonymous"
></script>

<!-- Stylesheet with SRI -->
<link
  rel="stylesheet"
  href="https://cdn.example.com/library.min.css"
  integrity="sha256-abc123..."
  crossorigin="anonymous"
/>
```

### Required attributes

**`integrity`** — the hash. Format: `<algorithm>-<base64-encoded-hash>`. Multiple hashes can be provided separated by spaces — the browser accepts the file if it matches any of them (useful for transitioning to a stronger algorithm):

```html
integrity="sha256-oldHash sha384-newHash"
```

**`crossorigin`** — required for cross-origin resources with SRI. Must be `"anonymous"` or `"use-credentials"`. Without this, the browser uses a no-CORS fetch and can't perform integrity checking on cross-origin resources.

---

## Computing the Hash

Using OpenSSL:

```bash
# Compute sha384 hash:
openssl dgst -sha384 -binary jquery.min.js | openssl base64 -A
# Output: abc123...

# Full integrity value:
echo "sha384-$(openssl dgst -sha384 -binary jquery.min.js | openssl base64 -A)"
```

Using Node.js:

```js
const crypto = require('crypto');
const fs = require('fs');

const content = fs.readFileSync('jquery.min.js');
const hash = crypto.createHash('sha384').update(content).digest('base64');
console.log(`sha384-${hash}`);
```

Most CDN services generate the `integrity` value for you — jsDelivr and cdnjs include it in their copy snippets.

---

## What Happens on Mismatch

If the downloaded file's hash doesn't match the expected hash:

- Browser refuses to execute the script / apply the stylesheet
- A console error reports the integrity violation: `Failed to find a valid digest in the 'integrity' attribute for resource 'https://cdn.../file.js'`
- A CSP violation report fires if the CSP includes `require-sri-for script`
- The page continues loading — the blocked resource is simply not applied

---

## `require-sri-for` CSP Directive

CSP can mandate SRI for all scripts or styles on a page:

```http
Content-Security-Policy: require-sri-for script; require-sri-for style
```

With this directive, any `<script>` or `<link rel="stylesheet">` without a valid `integrity` attribute is blocked — including same-origin resources. This enforces SRI usage as a policy.

---

## Limitations

**SRI doesn't protect against using a wrong version intentionally.** If you update the CDN URL to a new version without updating your hash, the browser blocks the new version (good). But if you intentionally update both the URL and hash to a malicious version, SRI does nothing — you vouched for the malicious file.

**SRI is for external content.** For files on your own origin, you have server-side control — use HTTPS and proper access controls instead. SRI is specifically for third-party trust chains.

**Hash must be kept current.** If the CDN updates a file at the same URL (e.g., a patch release at a non-versioned URL), your hash becomes invalid and the browser blocks it. Always use versioned URLs with SRI.

**SRI and dynamic content.** SRI only works for static files — files whose content is known at HTML-write time. It doesn't work for dynamically generated content.

---

## SRI in Build Pipelines

Webpack and Vite can generate SRI hashes for output bundles:

```js
// vite.config.js
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        // Content-hashed filenames (separate concern from SRI)
      }
    }
  },
  // Vite plugin for SRI:
  plugins: [
    // @vitejs/plugin-basic-ssl or vite-plugin-subresource-integrity
  ]
});
```

For self-hosted assets, content-addressed filenames (e.g., `main.a1b2c3d4.js`) are usually sufficient — the hash in the filename serves a similar cache-busting role, though it's not verified by the browser as a security measure.

---

## Interview Questions

**Q (High): What does SRI protect against and what are its limitations?**

Answer: SRI protects against CDN compromise — an attacker who gains write access to a CDN and modifies a served file cannot execute malicious code on your page, because the browser will compute a hash of the received file and reject it when it doesn't match the expected value in your HTML. The limitation is that SRI only verifies what was received matches what you expected when you wrote the HTML. It doesn't protect against: (1) you intentionally or mistakenly updating both the URL and the hash to a malicious version; (2) using a non-versioned CDN URL where the CDN legitimately updates the file (your hash breaks and the resource is blocked — an availability problem, not a security improvement); (3) the scenario where the attacker compromises your origin server (they can change the HTML and the hash). SRI is a supply chain protection for the CDN-to-browser link, not a general integrity system.

**Q (High): Why is `crossorigin="anonymous"` required for SRI on cross-origin resources?**

Answer: Without the `crossorigin` attribute, the browser makes a no-CORS fetch — a mode where the browser gets the resource but treats the response opaquely: JavaScript can't read it, and the browser can't verify integrity for cross-origin resources. CORS-mode fetches (triggered by `crossorigin="anonymous"`) get a transparent response that the browser can inspect. SRI integrity checking requires reading the response bytes to compute the hash. Additionally, the CDN must send `Access-Control-Allow-Origin: *` (or matching origin) for the CORS fetch to succeed — otherwise the browser will block the response for CORS reasons before it can even check the hash.

**Q (Medium): How do you generate an SRI hash for a file?**

Answer: Hash the exact bytes of the file using SHA-384 (the recommended algorithm for most cases) and base64-encode the result. Via OpenSSL: `openssl dgst -sha384 -binary file.js | openssl base64 -A` produces the base64 hash; prefix with `sha384-` for the integrity attribute. The hash is over the exact file bytes — even a single added space or newline change invalidates it. This is why SRI must use versioned, immutable CDN URLs. Most CDN copy-paste snippets (jsDelivr, cdnjs, Bootstrap's CDN) include the correct integrity value pre-computed. Never rely on computing it from a CDN URL without downloading and verifying the file yourself — if you fetch the hash from the same compromised CDN, you get the attacker's hash for their modified file.

**Q (Low): What does the CSP `require-sri-for` directive do?**

Answer: `require-sri-for script` in a CSP header mandates that every `<script>` element on the page must have a valid `integrity` attribute. Any script tag without one — including same-origin scripts — is blocked. This enforces SRI as a page-wide policy rather than relying on developers to add it manually. `require-sri-for style` applies the same requirement to `<link rel="stylesheet">`. This is useful for high-security pages that want to ensure every loaded resource is explicitly verified. The downside is maintenance overhead: every resource update requires a hash update. It's most practical for pages with few, infrequently updated external dependencies.

---

## Self-Assessment

- [ ] Explain what SRI protects against and the required attributes on a script tag
- [ ] Compute an SRI hash from the command line using OpenSSL
- [ ] Explain why `crossorigin="anonymous"` is required and what the CDN must return
- [ ] Describe what happens when a hash mismatch occurs
- [ ] Explain why non-versioned CDN URLs are incompatible with SRI

---
*Next: Clickjacking — X-Frame-Options and frame-ancestors CSP directive.*
