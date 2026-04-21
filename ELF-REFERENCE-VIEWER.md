# ELF Reference Viewer — Architecture

**Companion to ELF Format Specification v0.1**
**Status:** Design draft
**Date:** 2026-04-20

---

## 0. Purpose

This document specifies the architecture of the **ELF Reference Viewer**: one conforming implementation of the viewer requirements in ELF v0.1. It is not itself normative — the spec is. The goal is to demonstrate that the spec is implementable with current browser technology and to serve as a starting point for alternative viewers.

The Reference Viewer is published as two artifacts:

1. **Viewer site** (`https://viewer.elf-format.org/`) — a page that takes `?src=<url>` and renders the ELF standalone.
2. **Embed script** (`elf-embed.js`) — a drop-in `<script>` that replaces `<elf-embed>` custom elements on any host page with sandboxed iframes pointing at the viewer site.

Both artifacts share a common codebase.

---

## 1. High-level architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│  Host page (any origin)                                                │
│                                                                        │
│   <elf-embed src="https://example.com/handbook.elf">...</elf-embed>    │
│         │                                                              │
│         │  elf-embed.js replaces with:                                 │
│         ▼                                                              │
│   <iframe src="https://viewer.elf-format.org/embed?src=..."            │
│           sandbox="allow-scripts"                                      │
│           credentialless>                                              │
│   ┌────────────────────────────────────────────────────────────────┐  │
│   │  Viewer origin (cross-origin, sandboxed)                       │  │
│   │                                                                │  │
│   │   ┌───────────────┐   ┌─────────────────┐  ┌──────────────┐   │  │
│   │   │  Viewer shell │──▶│  ELF Loader     │──│ Cache (OPFS) │   │  │
│   │   │  (chrome UI)  │   │  — fetch        │  └──────────────┘   │  │
│   │   └───────┬───────┘   │  — parse        │                     │  │
│   │           │           │  — verify       │                     │  │
│   │           │           │  — extract      │                     │  │
│   │           │           └────────┬────────┘                     │  │
│   │           ▼                    ▼                              │  │
│   │   ┌─────────────────────────────────────────────┐             │  │
│   │   │  Inner iframe (srcdoc, isolated from shell) │             │  │
│   │   │  — runs index.html + runtime/main.js        │             │  │
│   │   │  — exposes window.elf (§6 of spec)          │             │  │
│   │   │  — wllama / webllm workers                  │             │  │
│   │   └─────────────────────────────────────────────┘             │  │
│   │                                                                │  │
│   └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

Two layers of isolation:

1. **Host ↔ Viewer**: cross-origin iframe with `sandbox`. The host cannot read viewer DOM; viewer cannot read host DOM.
2. **Viewer shell ↔ ELF runtime**: inner `srcdoc` iframe inside the viewer, run with a stricter CSP. This protects the viewer's own UI (back button, settings, cache manager) from a malicious ELF.

The inner-iframe design is what makes the ELF's stated "security model" actually enforceable: the ELF's runtime never touches the viewer's privileged context.

---

## 2. Components

### 2.1 Viewer shell

The outer viewer page. Responsibilities:

- URL parsing (`?src`, `?fingerprint`, `?embed=1`).
- Progress UI during load.
- Display of safety signals required by spec §8.7 (title, fingerprint, signatures, requested capabilities).
- User-accessible cache manager.
- Error UI for hash mismatches, unsupported capabilities, unsupported versions.
- The "terminate and clear" button.

Implemented as a small vanilla-JS or lightweight-framework app. **Does not** handle model inference or UI rendering of the ELF itself.

### 2.2 ELF Loader

A pure-JS module (`loader.ts`) with no DOM dependencies. Used by the shell and also usable in a Web Worker.

```typescript
interface ElfLoader {
  loadFromUrl(url: string, opts?: LoadOptions): Promise<LoadedElf>;
  loadFromBlob(blob: Blob, opts?: LoadOptions): Promise<LoadedElf>;
}

interface LoadedElf {
  manifest: Manifest;
  profile: 'single' | 'container';
  fingerprint: string;                 // sha256 of canonicalized manifest
  getResource(path: string): Promise<Blob>;
  getResourceUrl(path: string): Promise<string>;   // returns a blob: URL
  signatures: SignatureVerificationResult[];
  dispose(): Promise<void>;
}
```

Internal pipeline:

```
┌────────────────┐
│ Bytes (ArrayBuffer or Response.body stream)
└───────┬────────┘
        │
        ▼
┌────────────────┐       ┌────────────────┐
│ Magic-byte     │──ZIP─▶│ ZIP parser     │
│ dispatch       │       │ (fflate)       │
│                │──HTML─▶│ HTML parser    │
└────────────────┘       │ (DOMParser)    │
                         └────────┬───────┘
                                  │
                                  ▼
                         ┌────────────────┐
                         │ Manifest       │
                         │ validate       │
                         │ + canonicalize │
                         └────────┬───────┘
                                  │
                                  ▼
                         ┌────────────────┐
                         │ Capability     │
                         │ negotiation    │  ← aborts if unsatisfied
                         └────────┬───────┘
                                  │
                                  ▼
                         ┌────────────────┐
                         │ Resource index │  ← map of path → locator
                         │ (lazy)         │
                         └────────────────┘
```

The loader **does not** eagerly decode large resources. It builds an index mapping logical paths to locators (a ZIP central-directory entry, or a reference to an `elf-resource` script). Resources are extracted and hash-verified only when requested via `getResource`. For model files, this means the loader hands the locator to the Cache manager (§3) before any bytes are materialized, enabling dedup by manifest-declared hash alone.

### 2.3 Verifier

Two stages of verification, both in the loader:

**Manifest verification.** On parse, validate against a JSON Schema derived from spec §5. Reject on any validation failure. Compute the manifest fingerprint (canonicalized SHA-256 per spec §7.2).

**Resource verification.** When `getResource(path)` is called, stream the resource bytes through a `SubtleCrypto.digest` and compare against the manifest-declared `sha256`. Mismatch throws; throws are surfaced to the user by the shell as a clear error with the two hashes displayed.

For `elf-single`, per-resource `data-sha256` MUST be re-verified on every load (not just trusted from the attribute). Base64 decoding happens in a Web Worker to avoid blocking the main thread on large resources.

### 2.4 Runtime bindings

Separate modules, dynamically imported only when their binding is declared:

```
runtime-bindings/
├── wllama.js        - implements LanguageModel + EmbeddingModel over wllama
├── webllm.js        - implements LanguageModel over WebLLM
├── transformers.js  - implements EmbeddingModel over transformers.js
└── onnx.js          - implements EmbeddingModel over onnxruntime-web
```

Each binding exports:

```typescript
interface RuntimeBinding {
  readonly name: string;
  readonly supportedFormats: string[];
  readonly requiredCapabilities: string[];

  createLanguageModel?(
    weights: Blob,
    cfg: LanguageModelConfig,
  ): Promise<LanguageModelBackend>;

  createEmbeddingModel?(
    weights: Blob,
    cfg: EmbeddingModelConfig,
  ): Promise<EmbeddingModelBackend>;
}
```

`LanguageModelBackend` and `EmbeddingModelBackend` are the internal types the runtime orchestrator wraps to present the spec §6.3 `window.elf` surface. This separation means adding a new format (say, `safetensors` via a hypothetical WebGPU Safetensors runtime) only requires adding a new binding module; nothing else changes.

### 2.5 Runtime orchestrator

Loaded inside the inner iframe. Construction:

```javascript
// in the inner iframe, after srcdoc is loaded
import { ElfRuntime } from './runtime/main.js';

const manifest = JSON.parse(
  document.querySelector('script[type="application/elf-manifest"]').textContent
);

const resources = new ResourceMap();
for (const el of document.querySelectorAll('script[type="application/elf-resource"]')) {
  resources.register(el.dataset.path, el);
}

window.elf = new ElfRuntime(manifest, resources);
await window.elf.ready;
```

The orchestrator:

1. Reads the manifest.
2. Selects the runtime binding named in `language_model.runtime` (and `embedding_model.runtime` if different).
3. Verifies that the binding's `requiredCapabilities` are all present in the iframe context.
4. Extracts weight Blobs via `resources.get(path)` (verifying hash).
5. Calls `binding.createLanguageModel(...)` and `binding.createEmbeddingModel(...)`.
6. Wires the resulting backends into the §6.3 interface.
7. Loads `chunks.jsonl` into memory (streaming parser for large corpora).
8. For `index.type === "flat"`: memory-maps `vectors.bin` into a `Float32Array` view.
9. Fires `ready` event.

### 2.6 Retrieval module

Separate from the runtime bindings because it's binding-agnostic — it only needs the embedding function.

For `flat` index:

```javascript
async function retrieve(query, k = 5) {
  const [q] = await embeddingModel.embed([query]);       // Float32Array(dim)
  const { vectors, count, dim, normalized } = corpus;     // vectors: Float32Array

  // Dense top-k. For count < ~50k this is fine on main thread;
  // above that we push to a Worker.
  const scores = new Float32Array(count);
  for (let i = 0; i < count; i++) {
    let s = 0;
    const off = i * dim;
    for (let d = 0; d < dim; d++) s += q[d] * vectors[off + d];
    scores[i] = normalized ? s : s / (queryNorm * rowNorms[i]);
  }

  // Top-k via a min-heap of size k.
  return topK(scores, k).map(i => ({ ...chunks[i], score: scores[i] }));
}
```

For `hnsw` index: dynamically import an HNSW reader module and use it. Not required for 0.1 viewers but the interface is identical.

### 2.7 Cache manager

OPFS-backed, keyed by manifest-declared SHA-256:

```typescript
class ModelCache {
  async has(sha256: string): Promise<boolean>;
  async get(sha256: string): Promise<Blob | null>;
  async put(sha256: string, blob: Blob): Promise<void>;
  async evict(sha256: string): Promise<void>;
  async size(): Promise<number>;
  async list(): Promise<CacheEntry[]>;
}
```

Storage layout in OPFS:

```
/elf-cache/
├── manifest-fingerprints/
│   └── <fingerprint>.json     # metadata about ELFs the user has opened
├── resources/
│   └── <sha256>                # raw resource bytes
└── index.json                  # LRU + last-access metadata
```

Eviction policy: LRU with a default 10 GB cap, user-adjustable. When the user opens a second ELF that references the same model hash, `ModelCache.has()` returns true and the second load is ~instant.

---

## 3. End-to-end load sequence

This is the full path from `?src=<url>` to a functioning chat UI.

```
T=0      Viewer shell boots.
T=50ms   Parse query string. Extract src URL and optional fingerprint hint.

T=100ms  fetch(srcUrl, {mode: 'cors'})
         — Stream Response.body into the Loader. Display progress.

T=?      Loader reads first 8 bytes. Dispatches to ZIP or HTML parser.

         [ZIP path]
         ── Parse central directory.
         ── Extract manifest.json (small, read fully).

         [HTML path]
         ── DOMParser.parseFromString(text, 'text/html').
         ── Extract elf-manifest script's text content.

T=?+1s   Validate manifest against JSON Schema.
         Compute canonical fingerprint.
         If query had ?fingerprint=, compare. Mismatch → abort with error.

T=?+1s   Capability negotiation:
         — Check each `capabilities_required` against viewer's runtime checks.
         — WebAssembly: `typeof WebAssembly === 'object'`.
         — SIMD: `WebAssembly.validate(<8-byte simd module>)`.
         — WebGPU: `'gpu' in navigator`.
         — Threads: cross-origin isolation + SharedArrayBuffer check.
         — OPFS: `navigator.storage && 'getDirectory' in navigator.storage`.

T=?+1s   Model cache lookup:
         For each model declared in manifest:
           if cache.has(declared.sha256):
             skip extraction, mark "use cache"
           else:
             queue for extraction

T=?+2s   Build inner iframe srcdoc.
         — Inline the manifest (as elf-manifest script).
         — For each resource:
             — If NOT a model, or model not cached: extract, hash-verify,
               embed as elf-resource script OR streamed via postMessage.
             — If model IS cached: embed a stub that points the runtime
               at the cache via cache.get(sha256).
         — Inline the runtime bootstrap.

T=?+Xs   Create the iframe:
         iframe.srcdoc = constructed HTML;
         iframe.sandbox = 'allow-scripts';
         iframe.credentialless = true;
         iframe.allow = '';                   // no features
         Apply headers via the hosting response: CSP, COEP, COOP.

T=?+Xs   Inner iframe boots.
         — Parses DOCTYPE, meta, manifest script.
         — Bootstrap script imports runtime/main.js.
         — ElfRuntime constructor runs.
         — Binding loads (e.g., import('./runtime-bindings/wllama.js')).
         — Binding initializes wllama with the weights Blob.
         — Embedding model initializes.
         — chunks.jsonl streams into memory.
         — vectors.bin memory-mapped.
         — window.elf dispatches 'ready'.

T=?+Ys   UI (index.html) attaches listeners, renders, ready for input.
```

For an ELF with a 600 MB Llama model, cold load on a fast connection is dominated by the model fetch. Warm load (model already cached under its SHA-256, even from a *different* ELF) is seconds.

---

## 4. wllama integration specifics

The Reference Viewer's `wllama` binding uses the `@wllama/wllama` npm package. Sketch:

```javascript
// runtime-bindings/wllama.js
import { Wllama } from '@wllama/wllama';

const CONFIG_PATHS = {
  'single-thread/wllama.wasm': '/runtime/wllama/single-thread/wllama.wasm',
  'multi-thread/wllama.wasm':  '/runtime/wllama/multi-thread/wllama.wasm',
};

export const wllamaBinding = {
  name: 'wllama',
  supportedFormats: ['gguf'],
  requiredCapabilities: ['webassembly', 'simd'],

  async createLanguageModel(weightsBlob, cfg) {
    const wl = new Wllama(CONFIG_PATHS);
    await wl.loadModel([weightsBlob], {
      n_ctx: cfg.context_length,
      n_batch: 512,
      // n_threads: left default → auto
    });
    return {
      async *chat(messages, opts) {
        const prompt = applyChatTemplate(cfg.chat_template, messages);
        for await (const tok of wl.createCompletion(prompt, opts)) {
          yield { delta: tok, done: false };
        }
        yield { delta: '', done: true };
      },
      async tokenize(t)   { return wl.tokenize(t); },
      async detokenize(t) { return wl.detokenize(t); },
      async destroy()     { await wl.exit(); },
    };
  },

  async createEmbeddingModel(weightsBlob, cfg) {
    const wl = new Wllama(CONFIG_PATHS);
    await wl.loadModel([weightsBlob], {
      embeddings: true,
      pooling_type: cfg.pooling,  // 'mean' | 'cls' | ...
    });
    return {
      async embed(texts) {
        const out = [];
        for (const t of texts) {
          const v = await wl.createEmbedding(t, { normalize: cfg.normalize });
          out.push(new Float32Array(v));
        }
        return out;
      },
      async destroy() { await wl.exit(); },
    };
  },
};
```

The wllama WASM files (`single-thread/wllama.wasm`, `multi-thread/wllama.wasm`) are served from the viewer origin, not bundled inside every ELF. This is a deliberate choice: the runtime binding's machinery is the viewer's responsibility, not the ELF's. The ELF declares "I want the wllama binding at version >= 2.0"; the viewer provides it.

Exception: self-contained-archive mode. A publisher who wants an absolutely network-free ELF MAY include the wllama WASM files *inside* the ELF at `runtime/wllama/...`. The viewer prefers the viewer-provided version but will use the embedded one if specified via manifest `runtime.prefer_embedded: true`.

---

## 5. Weight loading via Blob (key optimization)

wllama's `loadModel` accepts `Blob[]` directly — no network fetch needed. This is what makes the cache work elegantly:

```
elf-resource (base64 in <script>) or ZIP entry
       ↓
Loader.getResource(path) — hash-verified Blob
       ↓
ModelCache.get(sha256)  ─── hit ───▶ existing OPFS File Blob
       │
       └── miss → ModelCache.put(sha256, blob) → OPFS File Blob
       ↓
wllama.loadModel([blob])
```

The cache hit path produces a Blob backed by an OPFS file, which wllama can stream directly — no duplication in memory.

---

## 6. Transport and hosting

### 6.1 Serving the viewer

The viewer itself is a static site. Required response headers:

```
Cross-Origin-Opener-Policy:   same-origin
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Resource-Policy: same-origin
Content-Security-Policy: (see §7)
```

COOP + COEP are required to enable `SharedArrayBuffer` and thus Wasm threads.

### 6.2 Serving an ELF

ELF files should be served with:

```
Content-Type:                 application/vnd.elf+zip       (or +html)
Cross-Origin-Resource-Policy: cross-origin
Access-Control-Allow-Origin:  *                             (or specific origin)
Cache-Control:                public, max-age=31536000, immutable
```

Immutable caching is safe because the ELF fingerprint (§7.2) changes if any content changes; publishers who update content publish to a new filename.

### 6.3 Embed script distribution

`elf-embed.js` is a tiny bundle (< 10 KB gzip target). Served from the viewer origin with CORP `cross-origin` so any host can load it.

---

## 7. CSP for the inner iframe

The viewer constructs the inner iframe's srcdoc and applies CSP via a meta tag. Default policy:

```
default-src 'none';
script-src 'self' 'wasm-unsafe-eval';
worker-src 'self' blob:;
style-src  'self' 'unsafe-inline';
img-src    'self' data: blob:;
font-src   'self' data:;
media-src  'self' blob:;
connect-src 'none';
frame-ancestors <viewer-origin>;
form-action 'none';
base-uri 'none';
```

For thin ELFs declaring `fetch_urls`, `connect-src` is replaced with the explicit list of origins. The viewer parses `fetch_urls` at manifest-validation time and refuses to load if any URL is not HTTPS or uses a scheme outside `https:`.

`'wasm-unsafe-eval'` is required by wllama (and any Wasm LLM runtime). General `'unsafe-eval'` is rejected — a runtime binding that requires it is refused with a clear error.

---

## 8. Failure modes and user-facing errors

Every failure surface has a defined user-facing message and, where applicable, a recovery path. Examples:

| Failure | User-facing message | Recovery |
|---|---|---|
| Not a valid ELF | "This file isn't a valid ELF (didn't match the expected magic bytes)." | — |
| Manifest schema fail | "This ELF's manifest is malformed: `<field>: <error>`." | — |
| Unsupported elf_version | "This ELF requires viewer version `<x>`. You're on `<y>`. Update at viewer.elf-format.org." | Link to updated viewer |
| Capability missing | "This ELF requires WebGPU, which isn't available in your browser." | Link to browser support matrix |
| Hash mismatch | "A resource in this ELF failed integrity check. Expected `<a>`, got `<b>`. This may indicate corruption or tampering." | Report link |
| Fingerprint mismatch (embed) | "This embed expected fingerprint `<a>` but the ELF has `<b>`. Refusing to load." | — |
| Signature fail | "Signature verification failed for identity `<did>`." | "Load anyway" (with warning) |
| Runtime unknown | "This ELF uses runtime `<name>`, which this viewer doesn't support." | List of supported runtimes |
| OOM during load | "This ELF's models exceed available memory." | Suggest smaller model fallback if declared |

Every error includes the ELF title and fingerprint so users can report reliably.

---

## 9. Test harness

The Reference Viewer is built alongside the conformance test suite (spec §12.3). The harness is a set of test ELFs plus a Playwright-driven test runner that:

1. Loads each test ELF in the viewer.
2. Asserts on expected behavior (loads successfully / fails with specific error / retrieves specific chunks for specific queries).
3. Exercises the interconversion tool (§9 of the spec) and re-runs the same assertions on the round-tripped output.

Test ELF categories:

- **Minimal** — smallest possible conforming ELF.
- **Full** — exercises every optional feature.
- **Adversarial** — malicious manifests (oversized, malformed, lying hashes), path traversal attempts, resource bombs.
- **Cross-profile** — same ELF in both profiles; output of operations must match.
- **Compatibility** — ELFs built against each registered runtime binding.

---

## 10. Non-goals for 0.1

Explicitly out of scope for the Reference Viewer at this stage:

- Streaming retrieval over a partially-downloaded ELF. (Possible later via HTTP Range requests into the ZIP central directory.)
- GPU fine-tuning inside the viewer.
- A native (non-browser) viewer. Feasible but separate project.
- Automatic conversion between weight formats (e.g., `gguf` → `mlc` on the fly). Offloaded to a separate build-time tool.
- Federated model cache across viewers on different origins. (The cache is per-viewer-origin by browser storage rules; this is intentional.)
