# Source Maps Strategy (dev vs. prod)

## Quick Reference

| Type | Build speed | Rebuild speed | Quality | Security risk | Use case |
|---|---|---|---|---|---|
| `eval-source-map` | Medium | Fast | High | Low (no file) | Dev |
| `cheap-module-source-map` | Fast | Fastest | Low (line only) | Low | Dev (large apps) |
| `source-map` | Slow | Slow | Highest | High | Prod (external) |
| `hidden-source-map` | Slow | Slow | Highest | Low | Prod (Sentry only) |
| `nosources-source-map` | Slow | Slow | Low | Lowest | Prod (public) |
| `false` / none | Fastest | Fastest | None | None | Prod (minimal setup) |

---

## What Is This?

Source maps are files that map compiled, minified, or transpiled code back to original source code. When an error occurs in production (pointing to minified `bundle.js:1:84726`), a source map translates that position back to your original `src/components/Button.tsx:42`.

```
Minified output:       function n(a,b){return a+b}n(1,2);
                                               ↑
Original source:       export function add(a: number, b: number) {
                         return a + b;
                       }
                       add(1, 2);
```

A source map stores this mapping as a VLQ-encoded table. Browser DevTools and error tracking tools (Sentry, Datadog) decode it automatically.

> **Check yourself:** What is the security risk of publishing a source map publicly alongside your minified bundle?

---

## Why It Matters

Without source maps, debugging production errors is nearly impossible — stack traces point to line 1 of a minified bundle. With the wrong source map strategy, you expose your entire source code to anyone who opens DevTools. The goal is to have high-fidelity error traces in error tracking without exposing source code publicly.

---

## Source Map Format

A `.map` file looks like:

```json
{
  "version": 3,
  "file": "bundle.js",
  "sources": ["src/utils/format.ts", "src/components/Button.tsx"],
  "sourcesContent": ["export function format...", "export function Button..."],
  "names": ["format", "Button", "onClick"],
  "mappings": "AAAA,SAAS,MAAM,CAAC,CAAS..."
}
```

The `mappings` field is a VLQ (Variable-Length Quantity) encoded string that maps every position in the generated file back to a position in a source file. The `sourcesContent` field optionally embeds the original source code inline — without it, the browser fetches the original files separately (if available).

---

## Linking Source Maps to Bundles

Two methods to attach a source map to a bundle:

```js
// 1. Source map URL comment at bottom of bundle.js — browser/DevTools loads it automatically
//# sourceMappingURL=bundle.js.map

// 2. HTTP header (for inline maps or alternative delivery)
SourceMap: /path/to/bundle.js.map
```

The browser only fetches the source map when DevTools is open — source maps don't affect page load performance for end users.

---

## Development Source Map Strategy

In development, the priority is **fast rebuild speed** and **accurate source mapping** for debugging.

### `eval-source-map` (webpack default for dev)

```js
// webpack.config.js
module.exports = {
  mode: 'development',
  devtool: 'eval-source-map',
};
```

Each module is wrapped in `eval()` with an inline data URL containing the source map. Rebuilds are fast because only changed modules regenerate their eval block. DevTools shows original source files.

```js
// Output structure:
eval("const add = (a, b) => a + b;\n//# sourceURL=webpack://app/./src/math.ts\n//# sourceMappingURL=data:application/json;base64,...")
```

### `cheap-module-source-map` (alternative for large apps)

Column information is omitted — error positions are accurate to the line but not the column. Build and rebuild are faster. Acceptable when your bundled output is minified beyond column usefulness anyway.

### Vite dev source maps

Vite uses esbuild for transforms, which generates source maps natively. In dev, Vite serves original files over native ESM — DevTools often shows the original TypeScript/JSX directly without needing traditional source maps because the browser's ESM resolution finds the original file.

---

## Production Source Map Strategy

In production, the priority is **error trace fidelity** for your team without **source code exposure** to the public.

### The security risk

If you publish a `.map` file alongside your bundle, anyone can download it and read your original source code verbatim — including business logic, authentication flows, API keys hardcoded by mistake, and proprietary algorithms. The `sourcesContent` field in the map contains the literal source files.

```
https://app.example.com/bundle.js
https://app.example.com/bundle.js.map  ← anyone can download this
```

### Option 1: No source maps (simplest, worst for debugging)

```js
// webpack
devtool: false

// Vite
build: { sourcemap: false }
```

Stack traces in error monitoring are unusable. Only appropriate for very small teams or when the codebase is not sensitive.

### Option 2: `hidden-source-map` (recommended for most teams)

```js
// webpack
devtool: 'hidden-source-map'
```

Generates `.map` files but does **not** add the `//# sourceMappingURL=` comment to the bundle. The browser cannot find the map automatically — DevTools cannot show original source. But error tracking tools like Sentry can be given the map files out-of-band (uploaded to Sentry's API at deploy time):

```bash
# Upload source maps to Sentry during CI/CD (not deployed to the server)
npx @sentry/cli sourcemaps upload \
  --release=$GIT_SHA \
  --url-prefix '~/static/js' \
  dist/
```

Sentry stores the maps internally and uses them to decode stack traces from error events — without the maps ever being publicly accessible.

### Option 3: `nosources-source-map`

```js
devtool: 'nosources-source-map'
```

Generates a map that contains position mappings (so stack trace line numbers are decoded) but omits `sourcesContent` (original code). DevTools shows the structure — "this error came from `src/Button.tsx:42`" — but won't show the source code itself if the original file isn't available. Lower fidelity than `hidden-source-map` with Sentry, but safer than a full map if you must publish.

### Option 4: Restrict map files by server config

Generate full source maps but only serve them to authenticated users or internal IPs:

```nginx
# Only serve .map files to internal network
location ~* \.map$ {
  allow 10.0.0.0/8;
  deny all;
}
```

This keeps full map quality while blocking public access, but requires infrastructure support and is easy to misconfigure.

---

## Sentry Source Map Upload Workflow

The standard for production error monitoring:

```
CI/CD pipeline:
  1. Build: generate bundle + .map files
  2. Upload maps: POST to Sentry API with release version
  3. Deploy: push bundle to CDN/server — WITHOUT .map files
  4. Error occurs: Sentry receives event with minified stack trace
  5. Sentry decodes: uses internally stored map to produce original trace
  6. Alert: you see exact file + line number in original TypeScript/JSX
```

```bash
# Vite plugin approach
# vite.config.ts
import { sentryVitePlugin } from '@sentry/vite-plugin';

export default defineConfig({
  plugins: [
    react(),
    sentryVitePlugin({
      authToken: process.env.SENTRY_AUTH_TOKEN,
      org: 'my-org',
      project: 'my-project',
    }),
  ],
  build: {
    sourcemap: true, // generate maps for upload
  },
});
```

---

## Source Map Types by Tool

```js
// Vite
export default defineConfig({
  build: {
    sourcemap: true,           // generates .map files (full)
    sourcemap: 'inline',       // embeds in bundle (large files)
    sourcemap: 'hidden',       // generates .map but no URL comment
  },
});

// Rollup
export default {
  output: {
    sourcemap: true,           // .map file
    sourcemap: 'inline',       // data URL in bundle
    sourcemapExcludeSources: true, // omit sourcesContent
  },
};

// TypeScript (tsc)
// tsconfig.json
{
  "compilerOptions": {
    "sourceMap": true,         // .js.map alongside .js
    "inlineSourceMap": false,  // don't embed
    "declarationMap": true,    // .d.ts.map for library consumers
  }
}
```

---

## Source Maps for Libraries

When publishing a library to npm:

- Include source maps so library consumers can debug through your code in DevTools
- Include `declarationMap: true` (TypeScript) so consumers can Cmd+Click to jump from a type to its source
- Do **not** include the original TypeScript sources unless you want to ship them (some teams do, some don't)

```json
// package.json — what to publish
{
  "main": "dist/index.cjs.js",
  "module": "dist/index.esm.js",
  "types": "dist/index.d.ts",
  "files": [
    "dist/",          // includes .map files
    "!dist/**/*.ts"   // exclude .ts source files if you don't want to ship them
  ]
}
```

---

## Gotchas

**1. Source maps inflate CI artifact size**
A `.map` file is typically 3–10x the size of the bundle. On a 1MB app, you might have 5–8MB of maps. Don't upload them to your CDN or include them in the main deployment artifact — upload to Sentry, then discard.

**2. `eval-source-map` breaks CSP**
Content Security Policy rules that block `eval()` (common: `script-src 'self'`) break `eval-source-map` in development. Use `cheap-module-source-map` or configure a CSP exception for dev environments.

**3. Source map quality depends on all transforms in the chain**
If your build pipeline is: TypeScript → Babel → webpack, each step must pass source maps to the next. A misconfigured step breaks the chain, and the final map points to intermediate transformed code (Babel output) rather than your original TypeScript. Use `source-map-loader` in webpack to consume TypeScript's source maps as input.

**4. Source maps can expose secrets**
If an API key or secret was ever in source code (even momentarily), the source map embeds it. Rotate the credential. Don't rely on "it's in the map file, not the bundle" as a security measure.

**5. Large source maps slow down Sentry symbolication**
Very large source maps (multi-MB) slow down Sentry's symbolication step. Prefer uploading per-chunk maps rather than one massive inlined map.

---

## Interview Questions

**Q (High): Why are source maps a security risk in production, and what is the recommended strategy?**

Answer: Source maps contain `sourcesContent` — the literal text of your original source files — as well as the position mappings. Publishing them publicly means anyone who downloads the `.map` file can read your entire codebase verbatim. The recommended strategy for production is `hidden-source-map` (webpack) or `sourcemap: 'hidden'` (Vite): generate the `.map` files during the build, but do not add the `//# sourceMappingURL=` comment to the bundles, so browsers cannot automatically find them. Upload the maps to your error tracking service (Sentry, Datadog) via their API during the CI/CD pipeline, then deploy the bundles without the map files. Your team gets full-fidelity stack traces in error monitoring, but the source maps are never publicly accessible.

**Q (High): How does Sentry use source maps if they're not deployed to your server?**

Answer: At deploy time, you upload the source maps to Sentry's API and associate them with a release identifier (typically the git commit SHA or a build ID). Sentry stores them internally. At error collection time, Sentry receives a JavaScript error event containing a minified stack trace — file name, line number, column in the minified bundle. Sentry matches the stack frame against the stored source map for that release, decodes the VLQ position mapping, and translates the minified location to the original file name and line number. The decoded stack trace is what's shown in the Sentry UI. The source maps never need to be on your production server — they're only needed by Sentry's symbolication service.

**Q (Medium): What is `eval-source-map` and why can't you use it in production?**

Answer: `eval-source-map` wraps each module's compiled code in an `eval()` call with an inline data URL containing the source map. This makes rebuilds very fast — only changed modules regenerate their eval block — and gives accurate source positions in DevTools. It can't be used in production for two reasons: (1) `eval()` is blocked by CSP `script-src` policies that don't include `'unsafe-eval'` — most security-conscious production apps have this policy; (2) the inline maps add significant file size (maps are embedded in the bundle rather than external files). For production, use external `.map` files with `source-map` or `hidden-source-map`.

**Q (Medium): What does `sourcesContent` do in a source map, and when would you omit it?**

Answer: `sourcesContent` is an array in the source map JSON that contains the full text of each original source file. When DevTools loads a source map with `sourcesContent`, it can display the original file without fetching it from the server — the code is embedded. When `sourcesContent` is omitted, DevTools still knows the file names and line numbers but must fetch the original files (which usually aren't available in production). Omit `sourcesContent` for `nosources-source-map` (webpack) when you want DevTools to show file names and line numbers in traces without exposing actual code — useful when source maps must be public but the code is sensitive. Sentry works best with `sourcesContent` included (it uses it for code context in error reports), so if you're using Sentry with private maps, keep `sourcesContent`.

**Q (Low): Why does `eval-source-map` conflict with Content Security Policy?**

Answer: `eval-source-map` wraps each module in `eval("... //# sourceMappingURL=data:...")`. Any Content Security Policy that includes `script-src` without `'unsafe-eval'` will block these eval calls, causing a CSP violation. This is why `eval-source-map` is development-only — most production CSP configurations block `eval`. In development with a strict CSP (e.g., when testing CSP locally), switch to `cheap-module-source-map` or `source-map`, which use file-based maps rather than eval wrappers.

---

## Self-Assessment

- [ ] Explain the security risk of deploying `.map` files publicly and how `hidden-source-map` mitigates it
- [ ] Describe the Sentry source map upload workflow and why maps don't need to be on the production server
- [ ] Explain why `eval-source-map` conflicts with CSP and what to use instead in dev with strict CSP
- [ ] Describe what `sourcesContent` contains and when to omit it
- [ ] Name the correct webpack `devtool` value for: (a) fast dev rebuilds, (b) production with Sentry, (c) production without Sentry

---
*Next: Polyfill Strategy — differential serving, browserslist, and core-js to ship modern JS without penalizing modern browsers.*
