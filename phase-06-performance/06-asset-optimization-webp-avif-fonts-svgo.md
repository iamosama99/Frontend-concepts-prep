# Asset Optimization: WebP/AVIF, Font Subsetting, SVGO

## Quick Reference

| Asset type | Key technique | Typical savings | Tool |
|---|---|---|---|
| JPEG/PNG | Convert to WebP | 25–35% | `sharp`, `imagemin-webp`, Squoosh |
| WebP | Convert to AVIF | 20–50% over WebP | `sharp`, `libavif`, Squoosh |
| PNG with transparency | WebP or AVIF | 30–60% | `sharp` |
| Web fonts | Subset to used characters | 50–90% | `pyftsubset`, `glyphhanger`, `fonttools` |
| Web fonts | WOFF2 compression | 30–40% over TTF | `woff2_compress` |
| SVG | Remove metadata, optimize paths | 20–50% | `svgo` |
| Images | Responsive `srcset` | 50–80% for small screens | `<img srcset>` + `sizes` |

---

## What Is This?

Asset optimization is the process of reducing the byte size of non-JavaScript resources — images, fonts, and SVGs — to the minimum required to deliver the intended visual quality. These assets often dominate the network payload; on image-heavy pages, images alone can account for 60–80% of total bytes transferred. Optimizing them is frequently the highest-leverage performance improvement available.

Unlike JavaScript, these assets can often be compressed dramatically (50–90%) without any change in visual fidelity — just by choosing the right format and stripping unnecessary metadata.

---

## Why Does It Exist?

Network bandwidth is the primary bottleneck for users on mobile connections. A 2 MB JPEG that could be served as a 400 KB AVIF wastes 1.6 MB of data on every page load. Multiply by 1000 concurrent users and the cost is real: user frustration, higher bounce rates, and increased CDN costs.

The browser has historically lacked a universal modern image format. JPEG (1992) and PNG (1996) are universal but old. WebP (2010) and AVIF (2019) offer dramatically better compression but require progressive adoption via the `<picture>` element.

---

## Image Formats

### WebP

WebP is a modern image format developed by Google. It supports lossy and lossless compression, transparency, and animation. For most photographic content, WebP delivers the same perceived quality as JPEG at 25–35% smaller file size. For images with transparency (previously PNG-only), WebP is often 50–60% smaller.

Browser support is universal (all modern browsers since 2021).

```html
<!-- Use <picture> to serve WebP with JPEG fallback -->
<picture>
  <source srcset="/hero.webp" type="image/webp" />
  <img src="/hero.jpg" alt="Hero image" width="1200" height="600" />
</picture>
```

The browser uses the first `<source>` it supports. Older browsers that don't support WebP fall through to the `<img>` fallback.

---

### AVIF

AVIF is derived from the AV1 video codec and offers 20–50% better compression than WebP at equivalent visual quality — often 50–80% smaller than the equivalent JPEG. It supports HDR, wide color gamut, and lossless compression.

Encoding is slow (several seconds per image at high quality), making it impractical to generate at request time — always pre-generate AVIF assets at build time or via a CI pipeline.

Browser support: Chrome 85+, Firefox 93+, Safari 16.4+. Always provide a WebP or JPEG fallback.

```html
<picture>
  <!-- AVIF first — best compression -->
  <source srcset="/hero.avif" type="image/avif" />
  <!-- WebP fallback -->
  <source srcset="/hero.webp" type="image/webp" />
  <!-- JPEG final fallback -->
  <img src="/hero.jpg" alt="Hero" width="1200" height="600" loading="lazy" />
</picture>
```

**Processing with sharp (Node.js):**

```js
import sharp from 'sharp';

async function optimizeImage(inputPath, outputDir) {
  const base = path.basename(inputPath, path.extname(inputPath));

  // Generate WebP
  await sharp(inputPath)
    .webp({ quality: 80 })
    .toFile(`${outputDir}/${base}.webp`);

  // Generate AVIF
  await sharp(inputPath)
    .avif({ quality: 60 })  // AVIF quality is on a different scale; 60 looks similar to WebP 80
    .toFile(`${outputDir}/${base}.avif`);

  // Generate responsive sizes
  for (const width of [400, 800, 1200, 1600]) {
    await sharp(inputPath)
      .resize(width)
      .webp({ quality: 80 })
      .toFile(`${outputDir}/${base}-${width}w.webp`);

    await sharp(inputPath)
      .resize(width)
      .avif({ quality: 60 })
      .toFile(`${outputDir}/${base}-${width}w.avif`);
  }
}
```

---

### Responsive Images with `srcset` and `sizes`

A 1200px hero image served to a 375px mobile viewport wastes ~90% of the pixels. The browser downloads the full 1200px image, decodes it, and scales it down. Responsive images serve the correct size for the current device.

```html
<picture>
  <source
    type="image/avif"
    srcset="
      /hero-400w.avif   400w,
      /hero-800w.avif   800w,
      /hero-1200w.avif 1200w
    "
    sizes="(max-width: 600px) 100vw, (max-width: 1200px) 50vw, 33vw"
  />
  <source
    type="image/webp"
    srcset="
      /hero-400w.webp   400w,
      /hero-800w.webp   800w,
      /hero-1200w.webp 1200w
    "
    sizes="(max-width: 600px) 100vw, (max-width: 1200px) 50vw, 33vw"
  />
  <img
    src="/hero-800w.jpg"
    alt="Hero"
    width="800"
    height="400"
    loading="lazy"
    decoding="async"
  />
</picture>
```

**`sizes` attribute is critical.** Without it, the browser assumes the image will be 100vw (full viewport width) and selects a larger candidate than needed. `sizes` describes the rendered size of the image for different viewport conditions:

```
sizes="(max-width: 600px) 100vw, (max-width: 1200px) 50vw, 33vw"
```

- Viewport ≤ 600px: image is 100% of viewport width → browser picks the 400w candidate for a 375px screen
- Viewport ≤ 1200px: image is 50% of viewport width → browser picks 800w for a 900px screen  
- Otherwise: image is 33% of viewport → browser picks 400w for a 1200px wide grid layout

> **Check yourself:** Why does the browser use `sizes` rather than just reading the CSS `width` of the `<img>` element to determine which srcset candidate to pick?

---

### `loading="lazy"` and `decoding="async"`

```html
<!-- Below-the-fold images: lazy load + async decode -->
<img
  src="/product.webp"
  alt="Product"
  width="400"
  height="300"
  loading="lazy"
  decoding="async"
/>

<!-- LCP image: eager load + high priority (never lazy the LCP) -->
<img
  src="/hero.webp"
  alt="Hero"
  width="1200"
  height="600"
  loading="eager"
  fetchpriority="high"
  decoding="sync"
/>
```

`loading="lazy"` defers image download until the image is near the viewport. Never apply it to the LCP image — the LCP image must start loading as early as possible.

`decoding="async"` allows the browser to decode the image off the main thread, preventing image decoding from blocking rendering. For the LCP image, `decoding="sync"` ensures the image is ready before the next paint.

---

## Font Optimization

### WOFF2 Format

All modern fonts should be served as WOFF2. It is based on Brotli compression and delivers 30–40% smaller files than WOFF, and 50–60% smaller than TTF/OTF.

```css
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter.woff2') format('woff2');
  font-display: swap;
  font-weight: 100 900;  /* variable font — one file for all weights */
}
```

No WOFF fallback is needed for any browser released after 2015.

---

### Font Subsetting

A full Latin character set font contains 200–400 glyphs. If your app only uses English text (A–Z, a–z, 0–9, punctuation), you need at most 100 glyphs. Subsetting removes unused glyphs, reducing file size by 50–90% for languages with smaller character sets.

**Using `pyftsubset` (fonttools):**

```bash
# Install fonttools
pip install fonttools brotli

# Subset to Latin characters only
pyftsubset inter.ttf \
  --output-file=inter-subset.woff2 \
  --flavor=woff2 \
  --unicodes="U+0020-007F,U+00A0-00FF,U+0131,U+0152-0153,U+02BB-02BC,U+02C6,U+02DA,U+02DC,U+2000-206F,U+2074,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD"
```

**Using `glyphhanger` (automated):**

```bash
# Crawl your site and automatically determine which characters are used
npx glyphhanger https://yoursite.com --subset=*.ttf --formats=woff2
```

`glyphhanger` crawls your pages, finds all the text content, and generates a subset containing only the glyphs that actually appear on your site.

---

### Unicode Range for Fallback Loading

When serving multiple language subsets, `unicode-range` tells the browser to only download the font file if the page contains characters from that range. The English subset is downloaded for English pages; the Cyrillic subset is downloaded only when the page contains Cyrillic characters.

```css
/* Latin subset — downloaded for English content */
@font-face {
  font-family: 'Inter';
  font-style: normal;
  font-weight: 400;
  src: url('/fonts/inter-latin.woff2') format('woff2');
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+02BB-02BC, U+02C6, U+02DA, U+02DC, U+2000-206F;
}

/* Cyrillic subset — only downloaded if page uses Cyrillic characters */
@font-face {
  font-family: 'Inter';
  font-style: normal;
  font-weight: 400;
  src: url('/fonts/inter-cyrillic.woff2') format('woff2');
  unicode-range: U+0400-045F, U+0490-0491, U+04B0-04B1, U+2116;
}
```

Google Fonts uses this pattern — the CSS it serves includes multiple `@font-face` rules with `unicode-range`, and the browser only downloads the subsets it needs.

---

### Variable Fonts

A variable font encodes multiple weights and styles in a single file using a design space. Instead of shipping separate files for Regular (400), Medium (500), SemiBold (600), and Bold (700), you ship one file that can render any value along the weight axis.

```css
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-var.woff2') format('woff2');
  font-weight: 100 900;   /* full weight range in one file */
  font-style: normal;
}

/* Use any weight — all served from one file */
h1 { font-weight: 700; }
p  { font-weight: 400; }
.label { font-weight: 550; }  /* fractional weights work with variable fonts */
```

A single variable font file is typically 20–40% larger than a single static weight file but 40–60% smaller than four static weight files combined. If you use three or more weights, a variable font is almost always smaller overall.

---

## SVG Optimization with SVGO

SVGs exported from Figma, Illustrator, or Sketch contain large amounts of unnecessary data: editor metadata, comments, precise decimal coordinates, redundant attributes, and unoptimized path data.

**SVGO** removes these automatically:

```bash
# Optimize a single SVG
npx svgo input.svg -o output.svg

# Optimize all SVGs in a directory
npx svgo -r -f ./icons -o ./icons-optimized
```

**SVGO output example:**

```svg
<!-- Before SVGO: 847 bytes -->
<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink"
  version="1.1" id="Layer_1" x="0px" y="0px" viewBox="0 0 24 24"
  style="enable-background:new 0 0 24 24;" xml:space="preserve">
  <style type="text/css">.st0{fill:#333333;}</style>
  <g id="arrow">
    <path class="st0" d="M12.00000,4.00000 L20.00000,12.00000 L12.00000,20.00000
      L10.58579,18.58579 L16.17157,13.00000 L4.00000,13.00000 L4.00000,11.00000
      L16.17157,11.00000 L10.58579,5.41421 Z"/>
  </g>
</svg>

<!-- After SVGO: 201 bytes -->
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24">
  <path fill="#333" d="M12 4l8 8-8 8-1.414-1.414L16.172 13H4v-2h12.172L10.586 5.414Z"/>
</svg>
```

**SVGO in a Vite/Webpack build pipeline:**

```js
// vite-plugin-svgr + svgo — optimize SVGs and import as React components
// vite.config.js
import svgr from 'vite-plugin-svgr';

export default {
  plugins: [
    svgr({
      svgrOptions: {
        plugins: ['@svgr/plugin-svgo', '@svgr/plugin-jsx'],
        svgoConfig: {
          plugins: [
            { name: 'preset-default' },
            { name: 'removeViewBox', active: false },  // keep viewBox for scaling
          ],
        },
      },
    }),
  ],
};
```

**SVGO gotchas:**
- SVGO's `removeViewBox` plugin removes the `viewBox` attribute, which breaks responsive SVGs. Disable it.
- Aggressive path merging can remove accessibility attributes. Check output SVGs manually.
- `cleanupIDs` renames IDs, which can break `<use>` references between SVGs. Disable if using SVG sprites.

---

## Image CDN Services

For production applications, image CDN services (Cloudinary, Imgix, Cloudflare Images) handle format conversion, responsive resizing, and optimization automatically at request time. You upload the source image once and request the format/size via URL parameters:

```
https://res.cloudinary.com/demo/image/upload/f_auto,q_auto,w_800/hero.jpg
```

- `f_auto`: serve AVIF, WebP, or JPEG based on browser `Accept` header
- `q_auto`: automatically determine the best quality/size trade-off
- `w_800`: resize to 800px width

This eliminates the need to pre-generate multiple formats and sizes in CI. The CDN caches the transformed image globally.

---

## Interview Questions

**Q (High): Why is AVIF better than WebP and when would you not use it?**
Answer: AVIF uses AV1 intra-frame encoding (the same algorithm that compresses AV1 video), which is significantly more efficient than WebP's VP8-based algorithm. At equivalent visual quality, AVIF is typically 20–50% smaller than WebP and 50–80% smaller than JPEG. It supports HDR, wide color gamut (P3), and alpha transparency. The reason not to use it as your sole format: encoding is slow (seconds per image) — impractical for user-uploaded or real-time content without pre-generation; browser support, while now broad, may not include all users if your audience includes older browsers (Safari < 16.4, Firefox < 93); some decoders still have quality issues on complex gradients. Always provide WebP and JPEG fallbacks via `<picture>`.
The trap: Claiming AVIF is universally better without mentioning encoding speed or the need for fallbacks.

**Q (High): How does the `sizes` attribute work, and what is the performance consequence of getting it wrong?**
Answer: The `sizes` attribute describes the layout size of the image for different viewport conditions, as CSS media queries paired with CSS length values. The browser evaluates these conditions from left to right and uses the first matching value to determine how large the image will be rendered. It then selects the srcset candidate whose width is closest to (but not smaller than) that size, accounting for the device pixel ratio. If you omit `sizes` or write `sizes="100vw"` for an image that is actually 33% of the viewport, the browser selects a candidate 3x larger than needed, wasting bandwidth. On a 360px mobile screen at 2x DPR, the browser requests a 720px image candidate — fine. But if `sizes` says 100vw when the image is 33vw, the browser requests a 2160px image candidate — a 9x bandwidth waste.
The trap: Thinking `srcset` alone is sufficient and `sizes` is optional.

**Q (High): What is font subsetting and how would you implement it for a production site?**
Answer: Font subsetting removes glyphs from a font file that are not used by the site's content. A full Latin font contains 300–400 glyphs; an English-only site may use 100. Removing unused glyphs reduces file size 50–90%. Implementation options: (1) Use `pyftsubset` from fonttools with explicit unicode ranges to generate a subset WOFF2 file at build time — precise and repeatable. (2) Use `glyphhanger` to crawl the production site, detect all characters used, and auto-generate the subset — catches edge cases but requires access to production content. (3) Use `unicode-range` in `@font-face` to split a font into language-specific subsets and let the browser download only the needed ones — this is what Google Fonts does. In production, combine subsetting (reduce file size) with WOFF2 format (best compression) and `unicode-range` (conditional loading).
The trap: Describing subsetting without mentioning WOFF2 format, or recommending `glyphhanger` without noting it needs access to real content.

**Q (Medium): Why should you never apply `loading="lazy"` to the LCP image?**
Answer: `loading="lazy"` defers image download until the image is near the viewport — typically within 1250–2500px below the current scroll position. For a hero image that is always in the initial viewport, this means the browser waits until layout is partially complete before initiating the fetch, delaying the image by hundreds of milliseconds. LCP measures when the largest visible element finishes rendering; any delay in starting the fetch directly adds to the LCP time. The LCP image should have `loading="eager"` (or no attribute, as eager is the default for above-the-fold images), `fetchpriority="high"` to outcompete other resource fetches, and a `<link rel="preload">` if it might be discovered late by the browser.
The trap: Assuming lazy loading all images is always better. Below-the-fold images benefit enormously; above-the-fold and LCP images are hurt.

**Q (Medium): A designer exports a 600 KB SVG icon set from Figma. What would you do with it before shipping?**
Answer: Run SVGO to remove editor metadata, optimize paths, and clean up attributes — typical savings are 30–60%. After SVGO: disable `removeViewBox` (needed for responsive scaling), disable `cleanupIDs` if the SVGs use internal `<use>` references or have accessibility IDs. Then decide how to ship them: for icons used throughout the app, compile them into a React component using SVGR (which runs SVGO automatically) so they are tree-shaken by the bundler — only imported icons appear in the bundle. For icons used once on a page, inline the SVG directly (no extra HTTP request). For large icon sets where many icons load on page, use an SVG sprite sheet — one HTTP request, all icons cached together, individual icons referenced via `<use href="#icon-name">`.
The trap: Only mentioning SVGO without discussing how to import/bundle the SVGs efficiently.

**Q (Low): What is a variable font and when is it a better choice than static font files?**
Answer: A variable font encodes multiple weights (and optionally widths, italics, optical sizes) in a single file using design axes that interpolate between defined instances. You get every weight from 100 to 900 in one file. Compared to static fonts: one variable font file is 20–40% larger than one static weight file. But if you use three or more weights, the variable font is smaller than the combined static weight files. The decision rule: use a variable font when you use ≥ 3 weights or any non-standard weight (like 550 for medium-bold). Use individual static files when you use only 1–2 specific weights and the variable font is significantly larger than the combined static files you need. Additional benefit: variable fonts enable smooth weight transitions via CSS `transition: font-weight` without swapping font files.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Explain the compression tradeoff between JPEG, WebP, and AVIF, and when each is appropriate
- [ ] Write a `<picture>` element serving AVIF with WebP and JPEG fallbacks plus responsive `srcset`
- [ ] Explain what `sizes` does and what the performance cost is of getting it wrong
- [ ] Describe font subsetting — what it removes, two tools that do it, and how `unicode-range` enables per-language subsetting
- [ ] Explain when to use a variable font vs. multiple static font files
- [ ] State two SVGO plugins you should disable and why

---
*Next: CSS Performance — asset optimization reduces image and font payloads; CSS performance covers how CSS itself can block rendering and what critical CSS extraction changes.*
