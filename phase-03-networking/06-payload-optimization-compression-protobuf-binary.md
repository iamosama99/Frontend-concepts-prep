# Payload Optimization: Compression, Protocol Buffers & Binary

## Quick Reference

| Technique | Compression ratio vs. JSON | Best for |
|---|---|---|
| Gzip | ~70% smaller | Wide compatibility, server default |
| Brotli | ~85% smaller than raw, ~15–20% smaller than gzip | Text responses (HTML, JS, JSON, CSS) |
| Protocol Buffers | 3–10x smaller than JSON | Typed internal service APIs, high-throughput |
| MessagePack | ~30% smaller than JSON | Binary JSON alternative, no schema needed |
| Binary (ArrayBuffer) | Domain-specific | Images, audio, sensor data, WebGL |

## What Is This?

The wire format and encoding of data affects how much bandwidth is used, how long the browser waits for responses, and how much CPU is spent serializing and deserializing. These costs compound at scale: a 200KB JSON response served to 10 million users per day costs 2TB of bandwidth daily. Compressing it to 40KB saves 1.6TB.

Payload optimization operates at two levels:
1. **Compression:** Apply an algorithm to reduce the byte count of an existing payload (gzip, Brotli)
2. **Encoding format:** Choose a more efficient representation from the start (protobuf instead of JSON, binary instead of text)

> **Check yourself:** Gzip and Brotli are both lossless compression algorithms applied to HTTP responses. If you apply Brotli to a protobuf payload, does the Brotli compression help as much as it would on a JSON payload? Why?

## Text Compression: Gzip vs. Brotli

Both gzip and Brotli are lossless compression algorithms used for HTTP responses. The server compresses the response body before sending; the browser decompresses before parsing.

### How They Work

Both exploit redundancy in data. Text (HTML, CSS, JSON, JavaScript) is highly redundant — common words, repeated patterns, consistent structure. Compression finds these patterns and encodes them more efficiently.

```http
# Server signals supported compression in response
Content-Encoding: gzip
# or
Content-Encoding: br

# Client signals what it accepts
Accept-Encoding: gzip, deflate, br
```

### Gzip

- **Released:** 1993, based on DEFLATE algorithm
- **Ratio:** Typically 60–80% reduction on text
- **CPU cost:** Low (fast to compress and decompress)
- **Support:** Universal — every browser, every server

Gzip is the safe default. If you're not doing anything, enable gzip on your server:

```nginx
# nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml;
gzip_min_length 256;   # don't gzip tiny responses (overhead > benefit)
gzip_comp_level 6;     # balance: 6 is good compression without excessive CPU
```

### Brotli

- **Released:** 2015, by Google
- **Ratio:** 15–25% better than gzip on the same content; overall ~80–85% reduction on text
- **CPU cost:** Higher compression time (offset by pre-compressing static assets at build time); decompression is fast
- **Support:** All modern browsers; requires HTTPS (the spec requires it)

Brotli achieves better ratios because it uses a larger sliding window (more context for pattern matching) and has a built-in dictionary of common web patterns (HTML tags, CSS property names, JavaScript keywords).

The key optimization for static assets: pre-compress with Brotli at build time (no runtime CPU cost), serve the pre-compressed file directly:

```
bundle.js          → 400KB original
bundle.js.gz       → 120KB (gzip, pre-compressed)
bundle.js.br       → 98KB  (brotli, pre-compressed)
```

```nginx
# Serve pre-compressed brotli for static files
location /static/ {
  brotli_static on;   # serves .br file if available and client accepts br
  gzip_static on;     # fallback to .gz if client doesn't support br
}
```

### Compression Limits

Compression works on repetition. Highly random data (already-compressed images like JPEG/PNG/WebP, encrypted data, already-compressed archives like .zip) doesn't compress further — applying gzip/Brotli adds overhead without size reduction. Configure your server to skip compression for these content types.

## Protocol Buffers (Protobuf)

Protocol Buffers are a binary serialization format and interface definition language developed by Google. Instead of encoding data as human-readable JSON text, protobuf encodes it as compact binary:

```protobuf
// product.proto — schema definition
message Product {
  uint64 id = 1;
  string name = 2;
  uint32 price_cents = 3;
  bool in_stock = 4;
  repeated string tags = 5;
}
```

The same data in JSON vs. protobuf:
```json
{"id":12345,"name":"Wireless Headphones","priceCents":8999,"inStock":true,"tags":["electronics","audio"]}
// JSON: 95 bytes (uncompressed)
```

Protobuf binary: ~35 bytes. Not human-readable, but compact.

### Why Protobuf Is Smaller

Protobuf eliminates JSON's verbosity:
- No field names in the wire format — fields are identified by their tag numbers (1, 2, 3 from the schema)
- Numbers encoded as variable-length integers (small numbers = fewer bytes)
- Booleans encoded as a single byte
- Strings only appear once (value, not "key": "value")

```
JSON {"inStock": true} = 16 bytes
Protobuf: field 4, wire type varint, value 1 = 2 bytes
```

### Protobuf and Code Generation

The `.proto` schema generates client and server code in any language:

```bash
# Generate TypeScript types and encode/decode functions
protoc --ts_out=src/generated product.proto

# Generated code handles serialization:
const product = Product.fromBinary(responseBuffer);
const buf = Product.toBinary(product);
```

Every field has a known type, so deserialization is faster (no type inference, no string-to-number coercion) and type safety is guaranteed.

### Browser Protobuf (gRPC-web / Connect)

Browsers can't use raw protobuf over HTTP/2 gRPC natively — they need either gRPC-web (with a proxy) or Connect (a modern gRPC-compatible HTTP/1.1 protocol):

```javascript
import { createPromiseClient } from '@connectrpc/connect';
import { createConnectTransport } from '@connectrpc/connect-web';
import { ProductService } from './generated/product_connect';

const client = createPromiseClient(
  ProductService,
  createConnectTransport({ baseUrl: 'https://api.example.com' })
);

const product = await client.getProduct({ id: BigInt(12345) });
// product is a fully typed Product object
```

## MessagePack

MessagePack is the "binary JSON" — same data model as JSON (objects, arrays, strings, numbers, booleans, null) but binary encoded. Unlike protobuf, no schema is required.

```javascript
import { encode, decode } from '@msgpack/msgpack';

// Serialize
const bytes = encode({ id: 42, name: 'Alice', active: true });
// bytes: Uint8Array, typically 30-40% smaller than the equivalent JSON string

// Deserialize
const obj = decode(bytes);
```

MessagePack is useful when you want to reduce payload size without introducing a schema step — drop-in replacement for `JSON.stringify/parse` with 30–40% size savings.

## Binary Data: `ArrayBuffer` and `Blob`

For truly binary data — images, audio samples, sensor readings, WebGL geometry — neither JSON (base64 encoding) nor text encoding is appropriate. Fetch can return raw binary:

```javascript
// Fetch binary as ArrayBuffer
const response = await fetch('/api/mesh-data');
const buffer = await response.arrayBuffer();

// Parse structured binary with DataView or TypedArrays
const view = new DataView(buffer);
const vertexCount = view.getUint32(0, true);  // little-endian
const x = view.getFloat32(4, true);
const y = view.getFloat32(8, true);

// Or use TypedArrays for uniform data
const floats = new Float32Array(buffer, 12);  // offset 12 bytes
```

Base64-encoding binary in JSON inflates the payload by ~33% and adds parsing overhead. For binary data (images, audio, geometry, sensor readings), use `ArrayBuffer` or `Blob` directly.

```javascript
// Don't base64-encode binary in JSON
{ "imageData": "iVBORw0KGgoAAAANSUhEUgAA..." }  // 33% overhead

// Fetch binary directly
const blob = await fetch('/api/image/42').then(r => r.blob());
const url = URL.createObjectURL(blob);
```

## Compression of Binary Formats

Binary formats (protobuf, images, WebGL geometry) are already more compact than their JSON equivalents. Applying Brotli/gzip to binary still helps for repetitive binary data, but the gains are smaller than on text:

- Brotli on JSON: 85% reduction (from 100KB to 15KB)
- Brotli on protobuf: 50–70% reduction (from 35KB to 11–17KB)
- Brotli on already-compressed images (JPEG, WebP): < 5% reduction (don't bother)

For high-frequency small protobuf messages (WebSocket frames, server-sent events), the compression overhead may exceed the savings. Measure before applying.

## Practical Priorities

1. **Enable Brotli compression** on your CDN and origin server for all text content (HTML, JSON, JS, CSS). This is the highest-leverage single change — often a free 20–30% improvement over gzip.
2. **Pre-compress static assets** at build time — no runtime CPU cost, maximum compression level.
3. **Don't compress** JPEG, PNG, WebP, WOFF2, ZIP — these are already compressed.
4. **Use protobuf** only if you control both client and server, need the type safety, and have measured that JSON is a bottleneck. The schema + codegen overhead isn't worth it for simple CRUD APIs.
5. **Use binary data** for binary payloads — never base64 in JSON.

## Gotchas

**Gzip/Brotli should have a minimum size threshold.** Compressing a 100-byte response adds HTTP header overhead and processing time that exceeds the size saving. Most servers have a configurable minimum (nginx defaults to 20 bytes — too low; 1KB is more practical).

**Brotli requires HTTPS.** The Brotli spec requires TLS. If your server or CDN serves over HTTP (redirect to HTTPS handled elsewhere), Brotli won't be used. In practice this is not a problem since all production apps should use HTTPS.

**Protobuf schema changes require care.** You can safely add new fields (old clients ignore unknown fields, new clients see defaults for missing fields). Changing field numbers or types without a migration breaks existing clients. Removing a field without reserving the tag number is a footgun — a future field might reuse that number and conflict with old clients.

**Content negotiation must work correctly.** The browser sends `Accept-Encoding: gzip, deflate, br`. The server must respond with `Content-Encoding: br` when it serves a Brotli response. If the server applies compression but sends the wrong (or no) `Content-Encoding` header, the browser attempts to parse compressed bytes as plain text — corrupt page.

**Streaming compressed responses and buffering.** Gzip/Brotli work on the full response body — they can't effectively compress a single streaming chunk. For Server-Sent Events or streaming responses, compression is applied per-chunk, which reduces effectiveness. Use chunked streaming OR heavy compression — not both.

## Interview Questions

**Q (High): What is the difference between gzip and Brotli, and when should you use each?**

Answer: Both are lossless HTTP content compression algorithms that reduce response body size before transmission. Gzip is older (1993), based on DEFLATE, with universal support across all browsers and servers. Brotli (2015, Google) achieves 15–25% better compression ratios than gzip on the same text content by using a larger sliding window and a built-in dictionary of common web patterns (HTML tags, CSS properties, JavaScript keywords). Brotli requires HTTPS. Use Brotli as the default for all modern production deployments — it offers meaningfully better compression at the cost of longer compression time (offset by pre-compressing static assets at build time). Keep gzip as a fallback for clients or infrastructure that doesn't support Brotli. The practical difference for a 200KB JavaScript bundle: ~70KB with gzip, ~60KB with Brotli. Across millions of requests, that's significant bandwidth savings.

The trap: Not knowing Brotli requires HTTPS, or not knowing that pre-compressing static assets eliminates Brotli's compression-time cost.

---

**Q (High): Why is Protocol Buffers more compact than JSON, and what are the trade-offs of using it in a web application?**

Answer: JSON encodes field names with every value (`"priceCents": 8999`) and represents all data as text (numbers as digit characters, booleans as the strings "true"/"false"). Protobuf eliminates field names from the wire format — fields are identified by compact integer tag numbers from the schema. Numbers are encoded as variable-length integers (1–5 bytes for most integers vs. their digit-count in JSON). The result is 3–10x smaller payloads for typical data. Trade-offs: protobuf requires a `.proto` schema definition and a code generation step — added build complexity. The binary format is not human-readable, making debugging harder (need a tool to decode). Both client and server must know the schema — it's not suitable for public APIs consumed by unknown clients. Browser support requires gRPC-web or Connect-web infrastructure. The size savings are most valuable for high-frequency calls or large datasets; for simple CRUD APIs with modest payloads, the complexity doesn't justify the gains.

The trap: Claiming protobuf is universally better. Name the schema/codegen overhead and the browser tooling requirement.

---

**Q (Medium): What does base64 encoding do to binary data in JSON, and what is the correct alternative?**

Answer: Base64 encoding converts binary bytes to ASCII characters using a 64-character alphabet. Three binary bytes become four ASCII characters — a 33% size increase. Embedding base64-encoded binary in JSON (e.g., an image as a string) adds this 33% overhead plus the JSON parsing step that treats the entire encoded value as a string before you can decode it. The correct alternative is to fetch binary data as an `ArrayBuffer` or `Blob` directly over a separate HTTP request or WebSocket frame. `fetch().then(r => r.arrayBuffer())` returns the raw bytes. Use `TypedArray` or `DataView` to parse structured binary. For images specifically, don't embed them in JSON at all — use `<img src="...">` or `URL.createObjectURL(blob)` with a direct binary fetch. Base64 in JSON is appropriate only as a last resort when a binary-capable transport isn't available.

The trap: Not knowing base64 inflates by 33%. This number is specifically useful in interviews about data efficiency.

---

**Q (Low): What is MessagePack and when would you use it instead of JSON or protobuf?**

Answer: MessagePack is a binary serialization format with the same data model as JSON — it supports objects, arrays, strings, numbers, booleans, and null — but encodes them as compact binary rather than human-readable text. Typical size reduction: 30–40% smaller than equivalent JSON. Unlike protobuf, MessagePack requires no schema definition or code generation — it's a drop-in replacement for JSON.stringify/parse. Use MessagePack when: you want payload savings without the schema overhead of protobuf, you have an existing JSON-based system and want a quick size win, the data structure varies between calls (making a fixed schema impractical). Avoid MessagePack when: the schema's type guarantees of protobuf matter for correctness, you're already using protobuf in the ecosystem (don't mix), or the non-human-readable format creates debugging pain without enough payload savings to justify it.

The trap: Not knowing MessagePack exists. It occupies a real position in the space between JSON and protobuf.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can explain why Brotli achieves better compression than gzip (larger window, built-in dictionary)
- [ ] Can explain why protobuf payloads are smaller than JSON (no field names, varint encoding)
- [ ] Can explain why base64-in-JSON adds 33% overhead and what the correct alternative is
- [ ] Can name when NOT to apply compression (JPEG, WebP, already-compressed formats)
- [ ] Can explain the trade-offs of protobuf (schema + codegen, not human-readable, better for internal APIs)
- [ ] Can explain what MessagePack is and when it's a better fit than protobuf

---
*Next: HTTP/2 Multiplexing & HTTP/3 (QUIC) — how the transport protocol changed to eliminate head-of-line blocking and how that changes frontend optimization strategies.*
