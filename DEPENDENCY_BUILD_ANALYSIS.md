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
| @agnai/web-tokenizers | 4.0 MB | Claude/LLaMA tokenization |
| @agnai/sentencepiece-js | 766 KB | SentencePiece tokenization |

These are the dominant contributors to install size. They include large WASM/binary artifacts that cannot be tree-shaken.

#### Deep Dive: sillytavern-transformers (53 MB)

**What it does:** Provides a Hugging Face Transformers.js-compatible pipeline for running ONNX ML models directly in Node.js via WASM. Used in `src/transformers.js` for five server-side AI tasks:

| Task | Default Model | Purpose |
|------|--------------|---------|
| `text-classification` | Cohee/distilbert-base-uncased-go-emotions-onnx | Sentiment/emotion analysis |
| `image-to-text` | Xenova/vit-gpt2-image-captioning | Image captioning |
| `feature-extraction` | Xenova/all-mpnet-base-v2 | Text embeddings for vector search |
| `automatic-speech-recognition` | Xenova/whisper-small | Speech-to-text (Whisper) |
| `text-to-speech` | Xenova/speecht5_tts | Text-to-speech synthesis |

**Internal breakdown:**
```
sillytavern-transformers (53 MB)
├── dist/ (47 MB)
│   ├── ort-wasm-simd.wasm          9.6 MB  ← ONNX Runtime WASM (SIMD)
│   ├── ort-wasm-simd-threaded.wasm 9.5 MB  ← ONNX Runtime WASM (SIMD+threads)
│   ├── ort-wasm.wasm               8.8 MB  ← ONNX Runtime WASM (baseline)
│   ├── ort-wasm-threaded.wasm      8.8 MB  ← ONNX Runtime WASM (threads)
│   ├── transformers.min.js.map     3.6 MB  ← Source map (not needed at runtime)
│   ├── transformers.js.map         2.4 MB  ← Source map (not needed at runtime)
│   ├── transformers.js             2.2 MB  ← Main library code
│   └── transformers.min.js         1.4 MB  ← Minified library code
├── node_modules/ (4.9 MB)         ← Bundled jimp (older version)
├── src/ (817 KB)                  ← Source code
└── types/ (484 KB)                ← TypeScript definitions
```

**Key issue — 37 MB of duplicated WASM:** The four `ort-wasm-*.wasm` files inside `sillytavern-transformers/dist/` are byte-for-byte identical to those in `onnxruntime-web/dist/`. This is because `sillytavern-transformers` depends on `onnxruntime-web@1.14.0` and also ships its own copies. At runtime, only the copies inside `sillytavern-transformers/dist/` are used (configured explicitly in `src/transformers.js:16`). The `onnxruntime-web` package's 66 MB is effectively dead weight.

**Key issue — 6 MB of source maps:** `transformers.js.map` and `transformers.min.js.map` are development artifacts not needed at runtime.

**Dependencies:** `onnxruntime-web@1.14.0`, `@huggingface/jinja`, `jimp@0.22.10` (bundled older version)

#### Deep Dive: onnxruntime-web (66 MB)

**Not used directly** — purely a transitive dependency of `sillytavern-transformers`. No code in `src/` imports it.

**Internal breakdown:**
```
onnxruntime-web (66 MB)
├── dist/ (63 MB)
│   ├── 4x WASM files (37 MB total) ← DUPLICATES of those in sillytavern-transformers
│   ├── ort.js / ort-web.js         ← Multiple JS entry points (3.7-3.8 MB each)
│   ├── 7x .map files               ← Source maps (~12 MB total)
│   └── Various minified builds      ← ES5, ES6, WebGL variants
├── lib/ (2.2 MB)                   ← TypeScript compiled sources
└── types/ (243 KB)
```

**The entire 66 MB package could theoretically be eliminated** if `sillytavern-transformers` bundled its own ONNX runtime (which it already does in `dist/`). This would require the upstream package to stop declaring `onnxruntime-web` as a dependency, or using npm overrides to alias it.

#### Deep Dive: tiktoken (23 MB)

**What it does:** OpenAI's BPE tokenizer used to count tokens for GPT models. Used in `src/endpoints/tokenizers.js` via `tiktoken.encoding_for_model(model)` to count tokens before sending requests to OpenAI APIs.

**Internal breakdown:**
```
tiktoken (23 MB)
├── tiktoken_bg.wasm    5.4 MB  ← Core WASM tokenizer engine
├── encoders/ (17 MB)           ← Token vocabulary files (3x duplication)
│   ├── o200k_base      2.3 MB × 3 formats (.json, .js, .cjs) = 6.9 MB  ← GPT-4o
│   ├── cl100k_base     1.1 MB × 3 formats = 3.3 MB                     ← GPT-4/3.5-turbo
│   ├── p50k_base       534 KB × 3 formats = 1.6 MB                     ← Codex
│   ├── p50k_edit       534 KB × 3 formats = 1.6 MB                     ← Edit models
│   ├── r50k_base       533 KB × 3 formats = 1.6 MB                     ← GPT-3
│   └── gpt2            533 KB × 3 formats = 1.6 MB                     ← GPT-2
├── lite/ (1.1 MB)              ← Lighter WASM variant
└── JS/TS files (~200 KB)
```

**Key issue — 3x format duplication in encoders:** Each encoder vocabulary is shipped in `.json`, `.js`, and `.cjs` formats. Only one format is used at runtime. This triples the encoders directory from ~5.5 MB to ~17 MB.

**Key issue — unused encoders:** If only GPT-3.5-turbo and GPT-4 models are tokenized, only `cl100k_base` and `o200k_base` are needed. The older `r50k_base`, `p50k_*`, and `gpt2` encoders (9.6 MB across all formats) may be unused.

#### Deep Dive: protobufjs (16 MB)

**Not used directly** — transitive dependency of `onnxruntime-web` (itself transitive via `sillytavern-transformers`). Used to deserialize ONNX model protobuf files.

**Internal breakdown:**
```
protobufjs (16 MB)
├── cli/ (13 MB)       ← CLI tools with own node_modules (@babel/parser, lodash, etc.)
│   └── node_modules/  ← Third copy of lodash (532 KB), @babel/parser (1.4 MB map), etc.
├── dist/ (2.2 MB)     ← Pre-built bundles
├── src/ (252 KB)      ← Source code
└── google/ (59 KB)    ← Proto definitions
```

**Key issue — 13 MB CLI tools:** The `cli/` directory contains protobuf code generation tools with their own `node_modules` (including yet another copy of lodash). These are only needed for `.proto → .js` compilation, never at runtime.

#### Deep Dive: @agnai/web-tokenizers & sentencepiece-js (4.8 MB)

**What they do:** Provide tokenization for non-OpenAI models (Claude, LLaMA, etc.) in `src/endpoints/tokenizers.js`.
- `@agnai/web-tokenizers` (4.0 MB): WASM-based tokenizer for Claude and other models using `Tokenizer` class
- `@agnai/sentencepiece-js` (766 KB): SentencePiece tokenizer for LLaMA-family models

These are relatively lean and well-justified for their functionality.

#### Deep Dive: vectra (323 KB) + transitive deps (~6.5 MB)

**What it does:** Local vector database for similarity search, used in `src/endpoints/vectors.js` to create and query `LocalIndex` instances for character memory/RAG features.

**Transitive dependency bloat:**
```
vectra (323 KB) depends on:
├── openai@3.x (2.0 MB)      ← NOT used by SillyTavern directly
├── axios (2.4 MB)            ← HTTP client
├── gpt-3-encoder (1.5 MB)   ← GPT-2 tokenizer (redundant with tiktoken)
├── cheerio (606 KB)          ← HTML parser
├── dotenv, uuid, yargs       ← Small utilities
└── json-colorizer            ← CLI formatting
```

SillyTavern only uses `vectra.LocalIndex` for local file-based vector storage. The `openai` package (2.0 MB) is declared as a dependency by vectra but is **not used** by SillyTavern (no direct imports). Similarly, `gpt-3-encoder` duplicates functionality already provided by `tiktoken`.

#### AI/ML Dependency Chain Summary

```
node_modules AI/ML (158 MB + 6.5 MB vectra deps = 164.5 MB)
│
├── sillytavern-transformers (53 MB) ← 5 ML tasks via ONNX
│   ├── 4x WASM files (37 MB) ← DUPLICATED in onnxruntime-web
│   ├── Source maps (6 MB) ← not needed at runtime
│   └── onnxruntime-web (66 MB) ← ENTIRELY UNUSED (dead transitive dep)
│       ├── 4x WASM files (37 MB) ← duplicates
│       ├── Source maps (12 MB)
│       └── protobufjs (16 MB)
│           └── cli/ (13 MB) ← dev tools, not needed at runtime
│
├── tiktoken (23 MB) ← OpenAI token counting
│   ├── WASM engine (5.4 MB)
│   └── Encoders (17 MB) ← 3x format duplication
│
├── @agnai/* (4.8 MB) ← Claude/LLaMA tokenization
│
└── vectra (323 KB) + transitive deps (6.5 MB) ← vector search
    └── openai, gpt-3-encoder, axios, cheerio ← mostly unused
```

**Total theoretical waste in AI/ML deps: ~98 MB**
- onnxruntime-web entirely (66 MB): WASM files duplicated, package unused directly
- protobufjs cli/ (13 MB): dev tools bundled in production
- tiktoken format duplication (11 MB): 3x .json/.js/.cjs
- sillytavern-transformers source maps (6 MB): dev artifacts
- vectra's unused transitive deps (~3.5 MB): openai, gpt-3-encoder

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
