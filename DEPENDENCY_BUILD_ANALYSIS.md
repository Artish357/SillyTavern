# Dependency Build Size Analysis

**Date:** 2026-02-26
**Total production dependencies:** 92 (direct), ~544 (installed/deduped)
**Total `node_modules` size:** 334 MB
**Webpack client bundle (`lib.js`):** 1,814 KiB minified / 570 KiB gzipped

---

## 1. Client-Side Bundle Breakdown (Webpack → `lib.js`)

The webpack build bundles 21 libraries from `public/lib.js` into a single production file.

| Library | Minified Size | % of Bundle | Notes |
|---------|--------------|-------------|-------|
| highlight.js | 936.0 KiB | 51.6% | **ALL 384 languages included** |
| moment | 292.0 KiB | 16.1% | **ALL 137 locales included** |
| lodash-es (via chevrotain) | 134.1 KiB | 7.4% | Transitive dep, full ESM build |
| yaml | 101.4 KiB | 5.6% | |
| showdown | 73.0 KiB | 4.0% | |
| handlebars | 72.8 KiB | 4.0% | |
| lodash (CJS) | 69.4 KiB | 3.8% | **Duplicate** — CJS build also bundled |
| @mozilla/readability | 33.4 KiB | 1.8% | |
| localforage | 28.5 KiB | 1.6% | |
| bowser | 25.8 KiB | 1.4% | |
| dompurify | 21.4 KiB | 1.2% | |
| @popperjs/core | 20.0 KiB | 1.1% | |
| diff-match-patch | 18.8 KiB | 1.0% | |
| fuse.js | 17.6 KiB | 1.0% | |
| @adobe/css-tools | 12.6 KiB | 0.7% | |
| seedrandom | 7.6 KiB | 0.4% | |
| chalk | 5.0 KiB | 0.3% | Unusual for a client bundle |
| morphdom | 4.8 KiB | 0.3% | |
| @iconfu/svg-inject | 4.8 KiB | 0.3% | |
| droll | 1.7 KiB | 0.1% | |
| slidetoggle | 1.6 KiB | 0.1% | |

### Key Findings — Client Bundle

1. **highlight.js is 51.6% of the bundle (936 KiB).** It imports all 384 language definitions. If only a subset of languages is needed, importing `highlight.js/lib/core` with selective `registerLanguage()` calls could reduce this to ~30-50 KiB.

2. **moment.js includes all 137 locales (292 KiB).** Webpack's `ContextReplacementPlugin` or `IgnorePlugin` can strip unused locales. Alternatively, migrating to `date-fns` or `dayjs` (~7 KiB) would provide similar functionality at a fraction of the size.

3. **Lodash is included TWICE (69.4 + 134.1 = ~203 KiB).** `public/lib.js` imports CJS `lodash`, while `chevrotain` brings in `lodash-es`. Both full builds end up in the bundle. Using `lodash-es` exclusively with webpack aliases could eliminate the CJS copy. Even better, importing only needed functions (`lodash-es/get`, `lodash-es/debounce`, etc.) enables tree-shaking.

4. **chevrotain is a heavy parser toolkit (~134 KiB).** It includes lodash-es as a dependency. If only basic parsing is needed, consider whether a lighter alternative exists.

5. **chalk in a browser bundle (5 KiB)** is unusual — this is a terminal color library. It will produce no visible effect in a browser. Consider removing it from the client bundle.

### Potential Client Bundle Savings

| Optimization | Estimated Savings |
|-------------|-------------------|
| highlight.js: selective language imports | ~880 KiB |
| moment → dayjs (or strip locales) | ~260 KiB |
| Deduplicate lodash/lodash-es | ~69 KiB |
| Remove chalk from client bundle | ~5 KiB |
| **Total potential savings** | **~1,214 KiB (67% reduction)** |

---

## 2. Server-Side Dependencies (node_modules disk impact)

These packages are NOT in the webpack client bundle but contribute to `node_modules` size and install time.

### AI/ML Dependencies — 158 MB (47% of node_modules)

| Package | Disk Size | Notes |
|---------|----------|-------|
| onnxruntime-web | 66 MB | Transitive via sillytavern-transformers |
| sillytavern-transformers | 53 MB | ML inference (includes WASM binaries) |
| tiktoken | 23 MB | OpenAI tokenizer (includes WASM) |
| protobufjs | 16 MB | Transitive via onnxruntime |

These are the dominant contributors to install size. They include large WASM/binary artifacts that cannot be tree-shaken.

### Image Processing — 24 MB (7% of node_modules)

| Package | Disk Size | Notes |
|---------|----------|-------|
| @jimp/* (18 packages) | 15 MB | Full image manipulation suite |
| @jsquash/* (transitive) | 8.7 MB | WASM image codecs |
| gifwrap (transitive) | 6.1 MB | GIF processing |
| image-size | 428 KB | Lightweight dimension detection |

The project imports 18 individual `@jimp` packages. If only a subset of image operations is needed, removing unused plugins would reduce this.

### Build Tools in Production — 32 MB

| Package | Disk Size | Notes |
|---------|----------|-------|
| typescript | 23 MB | Should be devDependency if not used at runtime |
| webpack | 6.2 MB | Used at runtime for on-the-fly bundling |

Note: `webpack` is listed as a production dependency because it's used at server startup to compile `lib.js` on-the-fly (via `src/middleware/webpack-serve.js`). This is intentional. However, `typescript` being in node_modules as a production install (likely transitive) adds 23 MB.

### Web Framework & Middleware — ~1.5 MB

Express and its middleware ecosystem are lightweight and well-justified:
express (278 KB), body-parser (471 KB), compression (112 KB), helmet (108 KB), cors (29 KB), cookie-parser (18 KB), cookie-session (145 KB), multer (70 KB), csrf-sync (35 KB), response-time (15 KB), rate-limiter-flexible (167 KB), host-validation-middleware (20 KB).

### Other Notable Dependencies

| Package | Disk Size | Notes |
|---------|----------|-------|
| web-streams-polyfill | 7.4 MB | Likely transitive; large for a polyfill |
| openai | 2.0 MB | Transitive via vectra |
| simple-git | 1.1 MB | Git operations |
| @agnai/* (tokenizers) | 4.8 MB | SentencePiece/tokenizer support |

---

## 3. Dependency Relationship Overview

```
node_modules (334 MB)
├── AI/ML stack (158 MB, 47%)
│   ├── sillytavern-transformers → onnxruntime-web (66 MB)
│   ├── tiktoken (23 MB, WASM)
│   └── protobufjs (16 MB, transitive)
├── Image processing (24 MB, 7%)
│   ├── @jimp/* (15 MB, 18 packages)
│   └── @jsquash/* + gifwrap (15 MB, WASM codecs)
├── Build tools (32 MB, 10%)
│   ├── typescript (23 MB)
│   └── webpack (6 MB, used at runtime)
├── Client bundle libs (pre-minification: ~28 MB, bundled to 1.8 MB)
│   ├── highlight.js (5.6 MB → 936 KiB minified)
│   ├── moment (4.4 MB → 292 KiB minified)
│   ├── lodash + lodash-es (2.5 MB → 203 KiB minified)
│   └── 18 other libraries (~15 MB → ~383 KiB minified)
├── Dev-only (@types, eslint) (~12 MB)
└── Everything else (~80 MB across ~480 packages)
```

---

## 4. Optimization Recommendations

### High Impact (Client Bundle)

1. **Selective highlight.js imports** — Import only needed languages instead of the full package. Saves ~880 KiB in the client bundle (51% of total).

2. **Strip or replace moment.js** — Use `webpack.IgnorePlugin` for `/^\.\/locale$/` to strip all locales, or migrate to `dayjs`. Saves ~260 KiB.

3. **Deduplicate lodash** — Alias `lodash` → `lodash-es` in webpack config so only one copy is bundled and tree-shaking can apply. Saves ~69 KiB.

### Medium Impact (Install Size)

4. **Audit @jimp plugin usage** — If not all 18 @jimp plugins are used, remove unused ones to reduce install by several MB.

5. **Evaluate `typescript` in production** — If it's only transitive, pinning or excluding it could save 23 MB of install size.

### Low Impact / Low Effort

6. **Remove `chalk` from client bundle** — Terminal colors have no effect in browsers. Saves 5 KiB.

7. **Consider `ContextReplacementPlugin` for moment** — If moment must stay, webpack plugin to include only used locales.

---

## 5. Build Configuration Notes

- **Bundler:** Webpack 5 in production mode with filesystem caching
- **Output:** Single ESM module (`lib.js`) via `libraryTarget: 'module'`
- **Tree-shaking:** Partially effective — 12 CJS modules prevent module concatenation (bailout messages in build output)
- **Source maps:** Disabled (`devtool: false`)
- **Code splitting:** None (single entry point, single output)
- **Compression:** Filesystem cache uses gzip; output is not pre-compressed (server handles gzip)
