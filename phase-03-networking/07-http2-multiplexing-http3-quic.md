# HTTP/2 Multiplexing & HTTP/3 (QUIC)

## Quick Reference

| Protocol | Transport | Key feature | Head-of-line blocking |
|---|---|---|---|
| HTTP/1.1 | TCP | One request per connection (pipelining broken in practice) | TCP + HTTP level |
| HTTP/2 | TCP | Multiplexed streams over one connection | TCP level (stream-level blocking) |
| HTTP/3 | QUIC (UDP) | Multiplexed streams, per-stream reliable delivery | None (streams are independent) |

## What Is This?

HTTP/1.1, HTTP/2, and HTTP/3 are successive versions of the protocol that moves data between browser and server. Each version changes how connections are managed and how requests are multiplexed — the impact on web performance is significant, and understanding the differences is necessary for making correct optimization decisions.

The central problem: HTTP/1.1 required a separate TCP connection per concurrent request. TCP connections are expensive (3-way handshake, slow start). Browsers worked around this by opening multiple parallel connections (typically 6 per domain), but this was a hack with its own costs. HTTP/2 fixed this with multiplexing. HTTP/3 fixed a remaining problem that HTTP/2's fix introduced.

> **Check yourself:** HTTP/2 allows unlimited streams over one TCP connection. HTTP/1.1 limited you to 6 parallel connections per domain. If HTTP/2 has one connection, why might it sometimes be slower than HTTP/1.1's 6 connections when there's packet loss?

## HTTP/1.1: The Baseline Problem

HTTP/1.1 (1997) uses TCP connections with one outstanding request per connection. Request A must get a response before Request B can be sent on the same connection. This is called "head-of-line blocking" at the HTTP layer.

**Workarounds browsers invented:**
- **6 parallel connections per domain:** Browsers open multiple TCP connections and multiplex requests across them. 6 connections × 1 request = 6 concurrent requests. More connections = more TCP handshakes, more slow-start ramp-up.
- **Domain sharding:** Serving assets from `cdn1.example.com`, `cdn2.example.com`, `cdn3.example.com` to bypass the 6-connection limit. More subdomains = more DNS lookups = more connections.
- **Resource bundling:** Concatenating many JS files into one, many CSS files into one. Fewer files = fewer HTTP requests.
- **CSS sprites:** Combining many small images into one large image. One request instead of 30.

All of these hacks have costs. Domain sharding adds DNS lookup time. Bundling means a change to one small file invalidates the entire bundle's cache entry. Sprites are maintenance nightmares.

HTTP/2 makes most of these hacks counterproductive.

## HTTP/2: Multiplexing

HTTP/2 (2015) keeps a single TCP connection per origin but allows multiple concurrent streams over that connection. Each stream is an independent request-response pair, and streams are interleaved freely.

### Binary Framing

HTTP/2 changes from text-based to binary-framed protocol. Requests and responses are split into **frames** — small units of data with headers identifying which stream they belong to:

```
Stream 1: [HEADERS frame][DATA frame][DATA frame]
Stream 3: [HEADERS frame][DATA frame]
Stream 5: [HEADERS frame][DATA frame][DATA frame][DATA frame]
```

All three streams' frames are interleaved over the single TCP connection. The receiving end reassembles frames by stream ID.

### HPACK Header Compression

HTTP/1.1 sends full headers with every request — `User-Agent`, `Accept`, `Cookie`, `Authorization` — on every request. For small assets, headers can exceed the payload size.

HTTP/2 uses HPACK compression: headers are compressed with a reference table shared between client and server. Frequently sent headers (`Content-Type: application/json`) are sent once and referenced by index on subsequent requests.

Result: header overhead drops from hundreds of bytes to a few bytes on repeated requests.

### Server Push

HTTP/2 allows the server to push resources to the browser before the browser requests them. The server can see the HTML request and immediately push the CSS and JS assets:

```
Client: GET /index.html
Server: PUSH_PROMISE /styles.css
Server: PUSH_PROMISE /bundle.js
Server: RESPONSE /index.html (with links to /styles.css, /bundle.js)
```

The browser has the CSS and JS before it even parses the HTML. In theory, this eliminates the "browser must parse HTML to discover assets" delay.

In practice: server push is poorly supported by CDNs, browsers have canceled pushed resources they don't want, and `<link rel="preload">` achieves similar goals more reliably. Server push has largely been abandoned in favor of `103 Early Hints` (a new mechanism that achieves the same goal more robustly).

### What HTTP/2 Makes Obsolete

- **Domain sharding:** Counterproductive — creates multiple TCP connections instead of the one multiplexed connection
- **Resource concatenation:** No longer needed for HTTP request reduction — but still useful for reducing total bytes
- **CSS sprites:** No request-count pressure anymore
- **Inlining small resources:** Smaller cache granularity (individual files cache independently) is now preferred

### HTTP/2's Remaining Problem: TCP Head-of-Line Blocking

HTTP/2's multiplexing solves HTTP-level head-of-line blocking (one request blocking another). But TCP itself still has head-of-line blocking.

TCP guarantees ordered, reliable delivery. If a TCP packet is lost in transit, the entire connection stalls waiting for that packet to be retransmitted — even if subsequent packets containing data for a completely different HTTP/2 stream have already arrived. Stream 3's data arrives but must wait for Stream 1's lost packet to be retransmitted before the receiver can process either.

On networks with packet loss > 1–2% (mobile networks, congested networks), HTTP/2's multiple streams can all be blocked by a single lost packet. On lossy networks, HTTP/1.1's multiple independent TCP connections can actually outperform HTTP/2, because a lost packet on one connection doesn't affect the others.

This is the problem HTTP/3 solves.

## HTTP/3 and QUIC

HTTP/3 (standardized 2022) replaces TCP with QUIC as the transport layer. QUIC is a modern transport protocol built on UDP.

### Why UDP?

UDP is connectionless and unreliable — it sends packets without guarantees of delivery or order. This sounds worse, but QUIC reimplements reliability on top of UDP with critical improvements:

- **Per-stream reliability:** Lost packets only block the stream they belong to, not all streams. Streams are truly independent.
- **Built-in encryption:** QUIC bakes TLS 1.3 into the protocol. The TLS handshake and the QUIC handshake happen simultaneously — 0-RTT on new connections, no round-trip penalty for adding TLS.
- **Connection migration:** QUIC connections are identified by a Connection ID, not by the IP:port tuple. When a phone switches from WiFi to cellular (different IP address), the QUIC connection survives — no reconnection, no interruption.
- **Reduced handshake latency:** QUIC + TLS 1.3 requires 1 RTT for new connections and 0 RTT for resumed connections (compared to TCP's 1 RTT + TLS's 1–2 RTT).

### 0-RTT Connection Establishment

This is one of HTTP/3's most significant practical benefits:

```
HTTP/1.1 new connection:
  TCP SYN → SYN-ACK → ACK (1 RTT)
  TLS ClientHello → ServerHello+Certificate → Finished (2 RTT)
  = 3 RTTs before first byte of data

HTTP/3 (QUIC) new connection:
  QUIC Initial (includes TLS ClientHello) → Server response (1 RTT)
  = 1 RTT before first byte of data

HTTP/3 (QUIC) resumed connection:
  0-RTT data sent with first packet (the client already has server keys)
  = 0 RTTs before first byte of data (with security trade-offs)
```

For mobile users who frequently switch networks and re-establish connections, this is a significant latency improvement.

> **Check yourself:** 0-RTT in QUIC allows sending data before the server confirms. What security concern does this introduce?

### HTTP/3 Current State

- **Browser support:** All major browsers support HTTP/3
- **Server support:** Cloudflare, Google, Fastly (CDN), nginx (via quiche or BoringSSL), LiteSpeed
- **Detection:** Browsers try HTTP/3 after seeing `Alt-Svc: h3=":443"` header from server
- **Fallback:** Browsers automatically fall back to HTTP/2 or HTTP/1.1 if QUIC is blocked by a firewall (some corporate firewalls block UDP port 443)

As a frontend engineer: if you're on a CDN that supports HTTP/3 (Cloudflare, Fastly), your users benefit automatically — no code changes required. HTTP/3 is a transport protocol; the HTTP semantics are unchanged.

## Performance Impact on Frontend Decisions

### HTTP/2's Impact on Bundling Strategy

HTTP/2 multiplexing means many small files over one connection is fine. But this doesn't mean unbundle everything:

- **Still bundle:** Avoid sending 1000 individual `node_modules` files — even with multiplexing, there's overhead per request (header frames, stream setup)
- **Smart chunking:** Split into medium-sized chunks (20–100 files instead of 1 or 1000). Granularity enables better caching without excessive request overhead
- **Tree shaking still matters:** Reducing total byte count is always correct regardless of protocol

### HTTP/2 Server Push vs. 103 Early Hints

`103 Early Hints` is the modern replacement for HTTP/2 server push:

```http
HTTP/1.1 103 Early Hints
Link: </styles.css>; rel=preload; as=style
Link: </bundle.js>; rel=preload; as=script

HTTP/1.1 200 OK
Content-Type: text/html
...
```

The server sends the 103 response before the HTML is fully processed. The browser immediately starts preloading the linked resources. When the 200 HTML arrives, the assets are already in flight. This works over HTTP/1.1 and HTTP/2, doesn't require protocol changes, and is much better supported than server push.

## Gotchas

**HTTP/2 with a single slow resource can still stall.** A slow database query that blocks rendering the HTML means the browser doesn't get the HTML quickly — and therefore doesn't request CSS and JS quickly. Multiplexing doesn't help if the critical bottleneck is one slow asset.

**QUIC (UDP) can be blocked by firewalls.** Many corporate firewalls block UDP port 443. HTTP/3 requires QUIC on UDP 443. Browsers handle this by falling back to HTTP/2 on TCP, but you may see HTTP/3 adoption rates lower in enterprise environments.

**0-RTT replay attacks.** QUIC's 0-RTT connection resumption sends data before the server can verify it isn't a replay. A network attacker could capture a 0-RTT packet and replay it. Sensitive mutating requests (POST, PUT, DELETE) should not be sent with 0-RTT. TLS 1.3 and QUIC include mechanisms to prevent replay, but this is a nuanced security concern.

**HTTP/2 multiplexing doesn't eliminate head-of-line blocking on lossy networks.** TCP packet loss blocks all HTTP/2 streams. On mobile networks where packet loss is common, HTTP/2 over TCP can underperform HTTP/1.1's multiple connections. This is specifically the problem HTTP/3/QUIC solves.

**Priority hints in HTTP/2:** HTTP/2 has a stream priority system allowing clients to signal which resources matter most. In practice, the priority implementation across servers was inconsistent. HTTP/3 simplifies priority signaling; the `importance` attribute on `<link>` and `fetchpriority` attribute on `<img>` and `<script>` give frontend control over priority.

## Interview Questions

**Q (High): What is HTTP/2 multiplexing and what problem does it solve compared to HTTP/1.1?**

Answer: HTTP/1.1 requires a separate request-response cycle to complete on a connection before the next can begin (head-of-line blocking). Browsers workaround this by opening up to 6 parallel TCP connections per domain — expensive in TCP handshake cost and slow-start ramp-up. HTTP/2 uses binary framing to split requests and responses into frames and multiplexes multiple streams over a single TCP connection. Stream 1, 3, and 5 can all be in-flight simultaneously, with their frames interleaved. This eliminates the need for domain sharding, reduces connection overhead, and enables fine-grained stream prioritization. The practical result: in a low-packet-loss environment (most desktop broadband), HTTP/2 is faster because one optimized TCP connection outperforms 6 cold connections. It also makes HTTP/1.1 optimization hacks (domain sharding, resource concatenation, CSS sprites) counterproductive.

The trap: Not knowing domain sharding becomes harmful under HTTP/2 (creates multiple connections instead of using the one optimized connection).

---

**Q (High): What is TCP head-of-line blocking in HTTP/2, and how does HTTP/3/QUIC solve it?**

Answer: HTTP/2 multiplexes streams over one TCP connection. TCP guarantees ordered, reliable delivery — if a packet is lost, TCP stalls the entire connection waiting for retransmission, even if packets belonging to other streams have already arrived. All HTTP/2 streams block waiting for the one retransmitted packet. This is TCP head-of-line blocking — HTTP/2 solved HTTP-level blocking but couldn't escape TCP's own blocking. HTTP/3 replaces TCP with QUIC, a protocol built on UDP. QUIC implements per-stream reliability: a lost packet only blocks the specific stream that packet belongs to. Streams are truly independent — stream 3's data doesn't wait for stream 1's retransmission. On networks with >1–2% packet loss (mobile, congested networks), this is a significant improvement. QUIC also includes TLS 1.3 built-in (eliminating TLS round-trips) and connection migration (connections survive IP address changes — important for mobile).

The trap: Stopping at "HTTP/2 uses one connection." The TCP head-of-line blocking problem is why this creates a new vulnerability that HTTP/3 addresses.

---

**Q (High): How should frontend bundling strategy change when targeting HTTP/2 vs. HTTP/1.1?**

Answer: HTTP/1.1's 6-connection-per-domain limit made large bundles attractive: fewer files = fewer HTTP requests. Domain sharding (serving assets from multiple subdomains) and CSS sprites were standard optimization techniques. HTTP/2 multiplexing eliminates the HTTP-level request bottleneck — many files over one connection is efficient. The strategy shift: instead of one giant bundle, use granular chunking. Vendor code (rarely changing) is separated from application code (frequently changing) so a code change only invalidates one small chunk's cache, not the entire bundle. Direct dependencies can be their own chunk. This improves cache hit rates. However: don't go to the extreme of one file per module (overhead per request still exists even under HTTP/2, and the request count for thousands of node_modules files is still excessive). A sensible target: 10–50 chunks for a large application. The most important bundling decision (tree shaking, dead code elimination) is unchanged by protocol version — reducing total bytes is always correct.

The trap: "HTTP/2 means you should unbundle everything." There's still overhead per stream; the middle ground of granular-but-not-atomized chunks is correct.

---

**Q (Medium): What is QUIC's 0-RTT connection resumption and what security trade-off does it introduce?**

Answer: QUIC with TLS 1.3 can establish a connection with 0 round-trips for resumed sessions (connecting to a server the client has connected to before). The client sends data in the very first packet, encrypted with session keys established in a previous connection. Compared to HTTP/1.1 (3 RTTs for TCP + TLS) or HTTP/2 (1–2 RTTs for TCP + TLS 1.3), 0-RTT dramatically reduces connection establishment latency — critical for mobile users who frequently reconnect. The security trade-off: replay attacks. If an attacker captures a 0-RTT data packet and replays it to the server, the server might process the request again (re-executing a POST, repeating a mutation). TLS 1.3 includes mechanisms to make 0-RTT data replay-detectable, but the detection isn't perfect. The mitigation: never send mutating requests (POST, PUT, DELETE) with 0-RTT. Idempotent GET requests are generally safe; non-idempotent mutations should use 1-RTT mode.

The trap: Not knowing the replay attack concern. 0-RTT sounds purely positive; the security qualification is what separates a complete answer from an incomplete one.

---

**Q (Low): What are `103 Early Hints` and why are they preferred over HTTP/2 server push?**

Answer: `103 Early Hints` is an HTTP status code that allows the server to send a preliminary response with `Link` headers (preload hints for CSS, JS, fonts) before the main `200` response is ready. The browser receives the 103 and immediately starts fetching the linked resources, while the server continues generating the HTML. When the 200 arrives, the critical assets are already in flight or cached. HTTP/2 server push achieved the same goal but had practical problems: CDNs widely didn't support it, browsers couldn't cancel pushes they didn't want (wasting bandwidth on already-cached resources), and the timing between push and request was complex to coordinate. `103 Early Hints` works over both HTTP/1.1 and HTTP/2, is widely supported by CDNs (Cloudflare, Fastly), lets the browser control whether to use the hint, and has much simpler semantics — it's an HTTP response, not a protocol-level push mechanism.

The trap: Describing server push as the solution for early asset delivery without knowing it was largely abandoned in favor of Early Hints.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain HTTP/1.1's head-of-line blocking and the 6-connection workaround
- [ ] Can explain HTTP/2 multiplexing: binary frames, streams, one TCP connection
- [ ] Can explain TCP head-of-line blocking and why it affects HTTP/2 but not HTTP/3
- [ ] Can explain QUIC: UDP-based, per-stream reliability, built-in TLS, connection migration
- [ ] Can explain how HTTP/2 changes bundling strategy (granular chunks vs. one bundle)
- [ ] Can name what 103 Early Hints does and why it replaced server push

---
*Phase 3 complete. Next phase: State Management & Data Flow — how state is categorized, where it lives, and the models (signals, observables, proxies) that make reactive UIs work.*
