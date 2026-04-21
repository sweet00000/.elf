# ELF Format Specification, version 0.1

**ELF — Embedded Language Format**
*Embedded content + local inference + declared UI contract.*

**Status:** Draft
**Editors:** Sweet et al.
**License of this document:** CC BY NC SA 4.0
**Date:** 2026-04-20

---

## 0. Introduction (non-normative)

An **ELF file** is a self-contained, offline-capable artifact that bundles (a) one or more machine-learning model weights, (b) a pre-embedded corpus of content, (c) an orchestration runtime, and (d) a user-facing interface, such that when opened in a conforming viewer the file delivers an interactive, retrieval-augmented experience — typically chat, but not limited to chat — entirely client-side.

The motivating use case is the same role PDF plays for static documents: any website can publish an `.elf`, any user can open it, and the experience works without a server. Unlike PDF, the experience is interactive, model-driven, and computed locally.

ELF is deliberately **container-agnostic at the surface**: the same logical artifact may be distributed as a single HTML file (the `elf-single` profile) or as a ZIP archive (the `elf-container` profile). Both profiles share a single manifest schema, runtime contract, security model, and conformance test suite. A lossless conversion between the two profiles is defined in §9.

### 0.1 Goals

- **Portability.** An `.elf` produced today should open in a conforming viewer in ten years.
- **Runtime neutrality.** The spec does not mandate a specific inference engine, model format, tokenizer, or UI framework.
- **Local-first.** A conforming `.elf` MUST be executable with no network access once loaded.
- **Cacheable.** Model weights MUST be deduplicable across ELF files by content hash.
- **Safe by default.** An embedding website MUST NOT be able to exfiltrate data from an ELF or inject code into it, and vice-versa.
- **Inspectable.** The manifest is human-readable JSON; contents are addressable by path and hash.

### 0.2 Non-goals

- Fine-tuning models inside the viewer. (Out of scope; may be a future extension.)
- Mandating a specific chat UI. The spec defines a runtime contract; any UI that honors it is conforming.
- Cryptographic DRM. Integrity and provenance are addressed; access control is not.

### 0.3 Document conventions

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY**, and **REQUIRED** in this document are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

---

## 1. Terminology

- **ELF file.** A file conforming to this specification in either the `elf-single` or `elf-container` profile.
- **Profile.** One of `elf-single` (a single HTML file) or `elf-container` (a ZIP archive). See §3 and §4.
- **Manifest.** The JSON document describing an ELF's contents, runtimes, capabilities, licenses, and provenance. See §5.
- **Resource.** A named, hash-addressed blob inside an ELF (a model file, a vector file, a chunk file, an asset, etc.).
- **Runtime contract.** The JavaScript API exposed to the UI layer by the ELF runtime. See §6.
- **Viewer.** A program that reads, verifies, and executes an ELF file.
- **Reference Viewer.** The viewer published at the canonical URL defined in §10; a specific implementation, not a normative requirement.
- **Host page.** A web page that embeds or links to an ELF via a viewer script.
- **Runtime binding.** A named implementation that satisfies a declared contract (e.g., `wllama`, `webllm`, `transformers.js`).
- **Capability.** A named browser or environment feature the ELF depends on (e.g., `webassembly`, `webgpu`, `simd`, `opfs`, `threads`).

---

## 2. File identification

### 2.1 Extension and MIME type

- File extension: **`.elf`**
- Media type: **`application/vnd.elf+zip`** for `elf-container`, **`application/vnd.elf+html`** for `elf-single`
- IANA registration is a deliverable of a later spec version.

### 2.2 Magic bytes

A conforming viewer MUST identify the profile of an ELF by inspecting its first bytes:

- If the file begins with the ZIP local file header signature `50 4B 03 04` and contains a valid `manifest.json` at its root, it is `elf-container`.
- If the file begins with an ASCII `<!DOCTYPE` or `<!doctype` declaration (optionally preceded by a BOM) and contains a `<script type="application/elf-manifest">` element with a valid manifest, it is `elf-single`.
- If neither applies, the file is not a conforming ELF.

Viewers MUST NOT rely on the file extension alone to determine the profile.

---

## 3. The `elf-container` profile

### 3.1 Container format

An `elf-container` file is a ZIP archive conforming to APPNOTE.TXT 6.3.9 or later. Implementations SHOULD use the STORE or DEFLATE compression methods for maximum interoperability; other methods MAY be used but MAY reduce portability. Encrypted ZIPs are NOT conforming.

### 3.2 Required structure

The following paths (relative to the archive root) have defined meaning:

```
manifest.json                REQUIRED
index.html                   REQUIRED (entry point for the UI)
runtime/                     REQUIRED (orchestration code)
  main.js                    REQUIRED
models/
  llm/                       conditional: REQUIRED if manifest declares a language model
  embed/                     conditional: REQUIRED if manifest declares an embedding model
corpus/
  chunks.jsonl               conditional: REQUIRED if manifest declares a corpus
  vectors.bin                conditional: REQUIRED if manifest declares a corpus with embeddings
  documents/                 OPTIONAL (raw source documents preserved for provenance)
  indices/                   OPTIONAL (ANN indices keyed by type, e.g., hnsw.bin)
assets/                      OPTIONAL (CSS, fonts, images, icons)
LICENSES/                    OPTIONAL (full license texts referenced by manifest)
```

Additional paths MAY be present; viewers MUST ignore paths not referenced by the manifest.

### 3.3 Path rules

- All paths in the manifest and archive MUST use forward slashes (`/`) as separators.
- Paths MUST NOT contain `..` segments, absolute prefixes, or symbolic links.
- Paths are case-sensitive.
- Maximum path length: 512 bytes UTF-8.

---

## 4. The `elf-single` profile

### 4.1 Container format

An `elf-single` file is a single HTML5 document that contains every resource of the logical ELF either inline or referenced by `data:` URI. The file MUST be openable by a browser directly from disk (`file://`) and MUST render a working experience without additional network fetches (modulo optional model-cache fallbacks, see §7.5).

### 4.2 Required structure

The file MUST include, in document order:

1. A `<!DOCTYPE html>` declaration.
2. A `<meta charset="utf-8">` element within `<head>`.
3. A `<meta name="elf-version" content="0.1">` element within `<head>`.
4. A `<script type="application/elf-manifest">` element containing the manifest JSON, placed before any resource scripts.
5. Zero or more `<script type="application/elf-resource" data-path="..." data-encoding="..." data-sha256="...">` elements, one per binary resource.
6. The UI shell as ordinary HTML.
7. A `<script type="module">` or `<script>` bootstrapping the runtime. This script MAY be the full runtime or MAY fetch the runtime from a declared CDN.

### 4.3 Resource encoding in `elf-single`

Each binary resource is embedded as a `<script type="application/elf-resource">` element. The `data-path` attribute gives the logical path (identical to the ZIP path in the container profile). The `data-encoding` attribute is one of:

- **`base64`** — the element's text content is standard Base64 (RFC 4648) of the raw resource bytes.
- **`base64url`** — the element's text content is Base64url (RFC 4648 §5).
- **`text`** — the element's text content is the resource's UTF-8 text (used for `manifest.json` only by convention; other text resources MAY use this).

The `data-sha256` attribute MUST be the lowercase hex SHA-256 of the decoded resource bytes. Viewers MUST verify every resource's hash before use.

Scripts with `type="application/elf-resource"` are not executed by the browser; they are inert data containers. The runtime extracts them by reading `document.querySelectorAll('script[type="application/elf-resource"]')`.

### 4.4 Size considerations

The `elf-single` profile is intended for small-to-medium ELF files (approximately up to 2 GB raw, taking into account the ~33% Base64 expansion). For larger payloads, publishers SHOULD use `elf-container`. Viewers MUST be able to handle `elf-single` files up to 4 GB encoded; behavior beyond that is implementation-defined.

---

## 5. The manifest

### 5.1 Format

The manifest is a UTF-8 JSON document. It MUST parse as a valid JSON object. Field ordering is not significant. Unknown fields MUST be preserved by tooling but MAY be ignored by viewers.

### 5.2 Top-level schema

```json
{
  "elf_version": "0.1",
  "id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
  "profile": "container",
  "title": "Example Knowledge Base",
  "description": "A short human-readable description.",
  "language": "en",
  "created": "2026-04-20T12:00:00Z",
  "authors": [
    { "name": "Sweet", "url": "https://example.com/sweet" }
  ],

  "language_model":   { ... },
  "embedding_model":  { ... },
  "corpus":           { ... },
  "ui":               { ... },
  "runtime":          { ... },

  "capabilities_required": ["webassembly", "simd"],
  "capabilities_optional": ["webgpu", "threads", "opfs"],

  "licenses":   [ ... ],
  "provenance": { ... },
  "signatures": [ ... ]
}
```

### 5.3 Core fields

| Field | Type | Req. | Description |
|---|---|---|---|
| `elf_version` | string | **R** | Semver string of the ELF spec version. `"0.1"` for this document. |
| `id` | string | **R** | Stable identifier. SHOULD be a `urn:uuid:` or `urn:sha256:` URN. |
| `profile` | string | **R** | `"container"` or `"single"`. Informational; the wire format is authoritative. |
| `title` | string | **R** | Human-readable title. |
| `description` | string | O | One-line description. |
| `language` | string | O | BCP 47 language tag describing the corpus's primary language. |
| `created` | string | **R** | ISO 8601 timestamp of build. |
| `authors` | array | O | List of `{name, url?, email?}`. |

### 5.4 `language_model`

Declares the optional language model. If present, the runtime MUST be able to produce text from a message list or prompt.

```json
"language_model": {
  "runtime": "wllama",
  "runtime_version": ">=2.0 <3",
  "format": "gguf",
  "path": "models/llm/model.gguf",
  "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "parameters": 1100000000,
  "quantization": "Q4_K_M",
  "context_length": 4096,
  "chat_template": "chatml",
  "default_sampling": { "temperature": 0.7, "top_p": 0.95, "max_tokens": 512 },
  "fallbacks": [
    { "runtime": "webllm", "format": "mlc", "path": "models/llm-mlc/", "sha256": "..." }
  ]
}
```

| Field | Type | Req. | Description |
|---|---|---|---|
| `runtime` | string | **R** | Runtime binding name (see §6.5). |
| `runtime_version` | string | O | Semver range the ELF was built against. |
| `format` | string | **R** | Weight format: `gguf`, `mlc`, `onnx`, `safetensors`, `ggml`, or an `x-` extension. |
| `path` | string | **R** | Logical path within the ELF. |
| `sha256` | string | **R** | Lowercase hex SHA-256 of the weight file. For directory-based formats (e.g., `mlc`), a Merkle root over entries sorted by path, with `sha256(path || 0x00 || sha256(content))` per entry, concatenated and hashed. |
| `parameters` | integer | O | Parameter count (for UI display). |
| `quantization` | string | O | Quantization method (for UI display). |
| `context_length` | integer | **R** | Maximum context length in tokens. |
| `chat_template` | string or object | **R** | Named template (`chatml`, `llama3`, `mistral`, `gemma`, `phi3`) or a Jinja2 template object `{"jinja": "..."}`. |
| `default_sampling` | object | O | Default inference parameters. |
| `fallbacks` | array | O | Alternative declarations for viewers that can't load the primary. |

### 5.5 `embedding_model`

```json
"embedding_model": {
  "runtime": "wllama",
  "format": "gguf",
  "path": "models/embed/model.gguf",
  "sha256": "...",
  "dimensions": 384,
  "pooling": "mean",
  "normalize": true,
  "max_input_tokens": 512,
  "fallbacks": []
}
```

| Field | Type | Req. | Description |
|---|---|---|---|
| `runtime` | string | **R** | Runtime binding name. MAY equal the `language_model.runtime`. |
| `format` | string | **R** | Weight format. |
| `path` | string | **R** | Logical path. |
| `sha256` | string | **R** | Hash. |
| `dimensions` | integer | **R** | Vector dimensionality. MUST match `corpus.dimensions`. |
| `pooling` | string | **R** | `mean`, `cls`, `max`, or `last`. |
| `normalize` | boolean | **R** | Whether vectors are L2-normalized at embed time. |
| `max_input_tokens` | integer | O | Max tokens for a single embedding call. |
| `fallbacks` | array | O | Same shape as above. |

### 5.6 `corpus`

```json
"corpus": {
  "chunks": "corpus/chunks.jsonl",
  "chunks_sha256": "...",
  "vectors": "corpus/vectors.bin",
  "vectors_sha256": "...",
  "vector_dtype": "float32",
  "vector_count": 12453,
  "dimensions": 384,
  "index": {
    "type": "flat",
    "path": null
  },
  "documents": "corpus/documents/",
  "chunking_strategy": {
    "method": "recursive-character",
    "chunk_size": 1024,
    "chunk_overlap": 128
  }
}
```

**`chunks.jsonl`** is a JSON Lines file. Each line is an object with at minimum `{"id": "<stable id>", "text": "<chunk text>"}` and MAY include `{"source": "<relative path under corpus/documents/>", "metadata": { ... }}`. Chunk order in the file MUST correspond one-to-one with vector order in `vectors.bin`.

**`vectors.bin`** is a raw packed array of `vector_count × dimensions` values in the declared `vector_dtype`, row-major, little-endian. Supported dtypes: `float32`, `float16`, `int8` (with declared `scale` and `zero_point` fields). No header, no footer.

**`index.type`** is one of `flat` (brute-force), `hnsw`, `ivf`, or an `x-` extension. If non-`flat`, `index.path` MUST be set.

### 5.7 `ui`

```json
"ui": {
  "entry": "index.html",
  "kind": "chat",
  "title_bar": true,
  "initial_message": "Ask me about the embedded documents.",
  "theme": { "primary": "#5A0EF8", "color_scheme": "auto" }
}
```

`kind` is a hint, not a constraint. Standard values: `chat`, `search`, `qa`, `summarize`, `translate`, `agent`, `blank`. Viewers MAY render `kind`-aware chrome (e.g., a toolbar label) but MUST NOT restrict what the UI can do based on `kind`.

### 5.8 `runtime`

```json
"runtime": {
  "entry": "runtime/main.js",
  "api_version": "0.1",
  "source_map": "runtime/main.js.map"
}
```

`api_version` is the version of the runtime contract (see §6). Viewers MUST refuse to execute an ELF whose `runtime.api_version` is unknown or incompatible with the viewer's supported set.

### 5.9 `capabilities_*`

Arrays of standardized capability tokens. Defined in this version:

- `webassembly` — baseline Wasm (MVP).
- `simd` — Wasm SIMD (fixed-width 128-bit).
- `threads` — Wasm threads (requires cross-origin isolation).
- `bulk-memory` — Wasm bulk memory operations.
- `webgpu` — the WebGPU API.
- `webgl2` — the WebGL 2 API.
- `opfs` — Origin Private File System.
- `fs-access` — File System Access API (user-visible directory handles).
- `shared-array-buffer` — `SharedArrayBuffer` availability.

Viewers MUST verify all `capabilities_required` before execution and MUST refuse to run an ELF whose required capabilities are unmet, surfacing a clear user-facing error.

### 5.10 `licenses`

```json
"licenses": [
  { "component": "language_model", "spdx": "Apache-2.0", "text": "LICENSES/apache-2.0.txt", "url": "..." },
  { "component": "embedding_model", "spdx": "MIT", "text": "LICENSES/mit.txt" },
  { "component": "corpus", "spdx": "CC-BY-4.0", "attribution": "..." },
  { "component": "runtime", "spdx": "MIT" },
  { "component": "ui", "spdx": "MIT" }
]
```

A conforming ELF MUST declare a license for every component present. The `spdx` field MUST be a valid SPDX identifier or `"LicenseRef-<tag>"` for custom licenses (which MUST then have `text` set).

### 5.11 `provenance`

```json
"provenance": {
  "builder": "elf-build/0.1.3",
  "build_host_os": "linux",
  "built_at": "2026-04-20T12:00:00Z",
  "source_documents": [
    { "path": "corpus/documents/handbook.pdf", "sha256": "...", "origin_url": "...", "retrieved_at": "..." }
  ],
  "reproducible_seed": null
}
```

Provenance is informational but strongly recommended. Viewers SHOULD display provenance to users on demand (e.g., a "Details" panel).

### 5.12 `signatures`

```json
"signatures": [
  {
    "algorithm": "ed25519",
    "public_key": "<base64>",
    "signed_fields": ["language_model.sha256", "embedding_model.sha256", "corpus.chunks_sha256", "corpus.vectors_sha256"],
    "signature": "<base64>",
    "identity_hint": "did:web:example.com"
  }
]
```

Signatures are OPTIONAL. When present, viewers MUST verify them and MUST display the signing identity to the user. A failed signature verification MUST be shown as a clear warning; viewers MAY still run the ELF but MUST surface the failure. Unsigned ELFs are valid.

---

## 6. Runtime contract

### 6.1 Overview

The runtime is JavaScript that, when loaded, exposes a global object `window.elf` satisfying the interface defined in this section. The UI (`index.html`) and any user code interact with the ELF exclusively through this object. The runtime implementation is free to use any backing library (wllama, webllm, transformers.js, ONNX Runtime Web, custom) as long as the surface is preserved.

### 6.2 Lifecycle

```
document parse
    ↓
runtime script loads
    ↓
window.elf = new ElfRuntime(manifest, resources)
    ↓
await window.elf.ready          // resolves when all models are loaded
    ↓
UI calls into window.elf.*      // normal operation
```

The runtime MUST emit lifecycle events on `window.elf` as a standard `EventTarget`:

| Event | `detail` | When |
|---|---|---|
| `manifest-loaded` | `{manifest}` | After manifest parse, before any model load |
| `resource-progress` | `{path, loaded, total}` | During resource load/decode |
| `model-loaded` | `{kind: 'llm' \| 'embed', path}` | After each model init |
| `ready` | `{}` | All declared models initialized |
| `error` | `{phase, message, cause}` | On any failure |

### 6.3 `window.elf` interface

```typescript
interface ElfRuntime extends EventTarget {
  readonly manifest: Manifest;
  readonly ready: Promise<void>;
  readonly capabilities: { [token: string]: boolean };

  // ---- Language model ----
  chat(messages: ChatMessage[], opts?: ChatOptions): AsyncIterable<ChatChunk>;
  complete(prompt: string, opts?: CompleteOptions): AsyncIterable<string>;
  tokenize(text: string): Promise<number[]>;
  detokenize(tokens: number[]): Promise<string>;

  // ---- Embedding model ----
  embed(texts: string[], opts?: EmbedOptions): Promise<Float32Array[]>;

  // ---- Retrieval ----
  retrieve(query: string, k?: number, opts?: RetrieveOptions): Promise<RetrievedChunk[]>;
  retrieveByVector(vector: Float32Array, k?: number): Promise<RetrievedChunk[]>;

  // ---- Corpus access ----
  getChunk(id: string): Promise<Chunk | null>;
  getDocument(path: string): Promise<Blob | null>;
  listDocuments(): Promise<string[]>;

  // ---- Resources ----
  getResource(path: string): Promise<Blob>;

  // ---- Lifecycle ----
  destroy(): Promise<void>;
}

type ChatMessage = { role: 'system' | 'user' | 'assistant' | 'tool'; content: string; name?: string };
type ChatChunk   = { delta: string; done: boolean; stop_reason?: string };
type Chunk       = { id: string; text: string; source?: string; metadata?: object };
type RetrievedChunk = Chunk & { score: number };
```

The runtime MUST throw a `DOMException` of name `NotSupportedError` if a method is called that the loaded models do not support (e.g., `embed` when no embedding model is declared).

### 6.4 Chat template handling

The runtime MUST apply the `chat_template` declared in the manifest to convert a `ChatMessage[]` into the tokens or prompt string the underlying model expects. Viewers MUST implement at minimum the templates: `chatml`, `llama3`, `mistral`, `gemma`, `phi3`. A Jinja2 template object `{"jinja": "..."}` SHOULD be supported when feasible; viewers that cannot evaluate Jinja2 MUST fail clearly with `NotSupportedError` rather than silently substituting.

### 6.5 Registered runtime bindings

The following binding names are registered by this version. Additional names MAY be registered through the process in §11.

| Name | Formats | Capabilities required |
|---|---|---|
| `wllama` | `gguf` | `webassembly`, typically `simd`, optionally `threads` |
| `webllm` | `mlc` | `webassembly`, `webgpu` |
| `transformers.js` | `onnx` | `webassembly`, optionally `webgpu` |
| `onnxruntime-web` | `onnx` | `webassembly`, optionally `webgpu` |

Runtime names not in this list MUST be prefixed with `x-` (e.g., `x-mycorp-engine`). Viewers MAY refuse to load ELFs declaring unknown runtimes.

### 6.6 Retrieval default behavior

If `retrieve(query, k)` is called and the manifest declares a `corpus.index` of type `flat`, the runtime MUST:

1. Call `embed([query])` to produce a query vector `q`.
2. Load `vectors.bin` into a typed array `V` of shape `[count, dim]`.
3. If `embedding_model.normalize` is true, assume both `q` and rows of `V` are unit-normalized and use dot product as the score; otherwise compute cosine similarity.
4. Return the top-`k` chunk IDs with scores, descending.

If `corpus.index.type` is non-`flat`, the runtime MUST use the declared index and MUST produce results equivalent up to approximation tolerances to the flat computation over the same vectors.

---

## 7. Hash, integrity, and caching

### 7.1 Resource hashes

Every binary resource declared in the manifest MUST carry a `sha256` field containing the lowercase hex SHA-256 of the raw resource bytes. Viewers MUST verify every hash before using the resource. A hash mismatch MUST be treated as a fatal error.

### 7.2 ELF-level identity

An ELF's canonical identity is the SHA-256 of its manifest JSON in **canonical form**: UTF-8 encoded, keys sorted lexicographically at every nesting level, no insignificant whitespace, numbers serialized per RFC 8785 (JSON Canonicalization Scheme). Viewers SHOULD expose this as the "ELF fingerprint" to users.

### 7.3 Model cache keying

Viewers SHOULD maintain a persistent cache of model weights keyed by the `sha256` declared in the manifest. When loading a model:

1. Compute the cache key: `sha256` from the manifest entry.
2. Check the viewer's cache. If a blob with that key exists and its SHA-256 verifies, use it.
3. Otherwise, extract the model bytes from the ELF, verify the hash, and store in cache under that key.

This protocol enables cross-ELF deduplication: two different ELFs referencing the same model weights share a single cached copy.

### 7.4 Cache storage

The recommended cache backend is OPFS (`capabilities_optional: ["opfs"]`). Viewers MAY use IndexedDB as a fallback. Viewers MUST provide users with a mechanism to inspect and clear the model cache.

### 7.5 Optional CDN fallbacks

A manifest entry MAY declare `fetch_urls: ["https://..."]` alongside the `path`. When an ELF is distributed without the model embedded (a "thin ELF"), viewers MAY fetch from the first URL whose content hash matches. Thin ELFs are a valid but distinct mode; they require network access and SHOULD be declared with `capabilities_required: [..., "network"]`. The `network` capability is reserved for this purpose.

---

## 8. Security model

### 8.1 Threat model

The spec assumes an adversarial setting:
- A malicious **ELF publisher** may attempt to exfiltrate data from the host page or the user's browser, execute drive-by code, or abuse the viewer's capabilities.
- A malicious **host page** may attempt to tamper with or spy on the ELF's state, steal embedded secrets, or manipulate the user's interaction with the ELF.
- A malicious **network observer** on thin-ELF downloads may attempt substitution (mitigated by `sha256` on all fetch_urls).

### 8.2 Execution isolation (normative)

A conforming viewer MUST execute an ELF's runtime, UI, and user-authored code inside a context isolated from the host page. The recommended mechanism is a cross-origin `<iframe>` with:

- `sandbox="allow-scripts"` (no `allow-same-origin`, no `allow-top-navigation`, no `allow-forms` unless the UI requires form submission which is strongly discouraged).
- A `Content-Security-Policy` header enforcing:
  - `default-src 'none'`
  - `script-src 'self' 'wasm-unsafe-eval'` (or `'unsafe-eval'` only if the runtime genuinely needs it; most don't)
  - `connect-src 'none'` for local-only ELFs; `connect-src <declared fetch_urls>` for thin ELFs
  - `img-src 'self' data: blob:`
  - `style-src 'self' 'unsafe-inline'`
  - `frame-ancestors <host origin>`
  - `form-action 'none'`
- `Cross-Origin-Embedder-Policy: require-corp` and `Cross-Origin-Opener-Policy: same-origin` when the ELF requires `threads` or `shared-array-buffer`.

Viewers MUST NOT grant the ELF iframe `allow-same-origin` against the host's origin.

### 8.3 Host ↔ ELF communication

The host page and the ELF iframe MUST communicate exclusively through `window.postMessage` using a small, schema-validated message set. The registered messages are:

| From | To | Type | Payload |
|---|---|---|---|
| host | elf | `elf/init` | `{manifestHash, origin}` |
| host | elf | `elf/prompt` | `{text, opts?}` |
| elf | host | `elf/ready` | `{manifest}` |
| elf | host | `elf/chat-delta` | `{delta, done}` |
| elf | host | `elf/error` | `{phase, message}` |
| elf | host | `elf/resize` | `{width, height}` |

Additional message types MUST be prefixed `x-`. Viewers MUST validate message shapes and MUST NOT forward untrusted messages verbatim.

### 8.4 What the host page MAY do

- Embed the ELF via the declared viewer script (§10).
- Observe `elf/ready`, `elf/chat-delta`, `elf/resize`, and `elf/error`.
- Send `elf/prompt` messages to programmatically drive the ELF.

### 8.5 What the host page MUST NOT do (enforced by isolation)

- Read the DOM or script state of the ELF iframe.
- Access the ELF's model cache.
- Exfiltrate chat transcripts (the ELF iframe's storage is partitioned).
- Inject code or styles into the ELF iframe.

### 8.6 What the ELF MUST NOT do

- Make network requests other than to declared `fetch_urls`. The CSP enforces this.
- Access cookies, localStorage, or IndexedDB of any origin other than its own partitioned storage.
- Attempt to navigate the top-level window.
- Execute `eval`-family operations beyond what the runtime explicitly requires. Runtimes that need `wasm-unsafe-eval` are permitted; general `unsafe-eval` is discouraged.

### 8.7 User-visible safety signals

Viewers MUST surface, without requiring the user to inspect developer tools:

- The ELF title, size, and fingerprint (§7.2).
- Whether the ELF is signed and, if so, by what identity (§5.12).
- What capabilities the ELF requested and received.
- Whether the ELF made any network requests (for thin ELFs) and their destinations.
- A one-click path to terminate the ELF and clear its state.

---

## 9. Profile interconversion

A tool named (conventionally) `elf-fuse` converts between profiles. Conversion MUST be lossless: the manifest, all resources, and all hashes MUST round-trip unchanged.

### 9.1 `container` → `single`

1. Read `manifest.json` from the ZIP; insert its text into a `<script type="application/elf-manifest">` element.
2. For each other resource `R` at path `P`:
   - Base64-encode `R`'s bytes.
   - Emit `<script type="application/elf-resource" data-path="P" data-encoding="base64" data-sha256="...">BASE64</script>`.
3. Emit the `<!DOCTYPE html>`, charset meta, and `elf-version` meta declarations.
4. Append the UI shell HTML (from `index.html`) **with its resource references rewritten** to use a runtime helper: `<img src="...">` becomes `<img data-elf-src="assets/logo.png">` or equivalent, and the runtime substitutes blob URLs at load time.
5. Append the runtime bootstrap script.

### 9.2 `single` → `container`

1. Parse the HTML; extract the `elf-manifest` script as `manifest.json`.
2. For each `elf-resource` script, decode per `data-encoding` and write to the declared `data-path`.
3. Reconstruct `index.html` by removing all `elf-manifest`, `elf-resource`, and bootstrap scripts; restore resource references.
4. Pack into a ZIP archive.

### 9.3 Equivalence

An `elf-fuse --roundtrip` operation (container → single → container, or vice versa) MUST produce an output whose manifest hash (§7.2) is identical to the input's.

---

## 10. Distribution and embedding

### 10.1 Canonical Reference Viewer

The Reference Viewer is published at `https://viewer.elf-format.org/` (reserved, pending domain acquisition by the spec editors). It accepts:

- `https://viewer.elf-format.org/?src=<url-encoded URL to .elf>` — standalone viewer.
- `https://viewer.elf-format.org/embed?src=<...>` — iframe-optimized embed view.

Viewers SHOULD, but are not required to, use this URL. A website MAY self-host a conforming viewer.

### 10.2 Drop-in embed script

A single-file script at `https://viewer.elf-format.org/elf-embed.js` MAY be included by host pages. It recognizes elements of the form:

```html
<elf-embed src="https://example.com/handbook.elf"
           height="600"
           fingerprint="sha256:abc..."></elf-embed>
```

And replaces them with a sandboxed iframe per §8.2. The `fingerprint` attribute, when present, MUST be verified; mismatch produces a clearly visible error.

### 10.3 Link behavior

A bare `<a href="handbook.elf">` link served with MIME `application/vnd.elf+zip` or `application/vnd.elf+html` SHOULD, in browsers with a registered ELF handler, open in that handler. In browsers without one, it SHOULD fall back to download. Specification of browser-native handling is a future work item.

---

## 11. Extensibility and versioning

### 11.1 Versioning

This document defines `elf_version: "0.1"`. Backward-incompatible changes bump the major version; additive changes bump the minor. Viewers MUST refuse to execute ELFs with a major version they do not implement, emitting a clear error directing the user to an updated viewer.

### 11.2 Extension fields

Any field name beginning with `x-` is reserved for non-standardized extensions. Tooling MUST preserve `x-` fields on round-trip but MAY ignore them at runtime.

### 11.3 Registration

New runtime bindings (§6.5), capability tokens (§5.9), and chat template names (§6.4) are registered by pull request against this document's repository. Registration requires: a written specification, a reference implementation, and interoperability tests that pass against the conformance suite.

---

## 12. Conformance

### 12.1 ELF conformance

An ELF is conforming at profile P, version V, if and only if:

- It parses per §3 (container) or §4 (single).
- Its manifest validates against the schema of §5.
- Every resource hash verifies.
- Every declared runtime and chat template is one registered at version V or a correctly-namespaced `x-` extension.
- It passes all applicable tests in the Version V conformance suite.

### 12.2 Viewer conformance

A viewer is conforming at version V if and only if:

- It correctly parses both profiles.
- It enforces §7 (integrity) and §8 (security).
- It implements all methods of the §6.3 runtime interface with the documented semantics for the runtime bindings and chat templates it advertises support for.
- Unsupported features produce the errors specified in §6.3, not silent fallbacks.
- It passes all applicable tests in the Version V conformance suite.

### 12.3 Conformance suite

The conformance suite is a set of ELF files plus a test harness, published alongside this specification in the reference repository. The suite exercises: manifest validation, hash verification, profile interconversion, chat template application, retrieval correctness, capability negotiation, signature verification, and security sandbox enforcement. The suite is versioned to match the spec version.

---

## Appendix A — Media type registration (informative)

Registration template for IANA, to be filed post-0.1 stabilization:

```
Name: elf+zip
File extension: .elf
Media type: application/vnd.elf+zip
Contact: <editor email>
Description: ELF (Embedded Language File) container profile.
```

And parallel for `application/vnd.elf+html`.

## Appendix B — Example minimal manifest (informative)

```json
{
  "elf_version": "0.1",
  "id": "urn:uuid:00000000-0000-4000-8000-000000000001",
  "profile": "container",
  "title": "Hello ELF",
  "created": "2026-04-20T12:00:00Z",
  "authors": [{ "name": "Anon" }],
  "language_model": {
    "runtime": "wllama",
    "format": "gguf",
    "path": "models/llm/tinyllama.gguf",
    "sha256": "0000000000000000000000000000000000000000000000000000000000000000",
    "context_length": 2048,
    "chat_template": "chatml"
  },
  "embedding_model": {
    "runtime": "wllama",
    "format": "gguf",
    "path": "models/embed/minilm.gguf",
    "sha256": "0000000000000000000000000000000000000000000000000000000000000000",
    "dimensions": 384,
    "pooling": "mean",
    "normalize": true
  },
  "corpus": {
    "chunks": "corpus/chunks.jsonl",
    "chunks_sha256": "0000000000000000000000000000000000000000000000000000000000000000",
    "vectors": "corpus/vectors.bin",
    "vectors_sha256": "0000000000000000000000000000000000000000000000000000000000000000",
    "vector_dtype": "float32",
    "vector_count": 42,
    "dimensions": 384,
    "index": { "type": "flat", "path": null }
  },
  "ui":      { "entry": "index.html", "kind": "chat" },
  "runtime": { "entry": "runtime/main.js", "api_version": "0.1" },
  "capabilities_required": ["webassembly", "simd"],
  "capabilities_optional": ["webgpu", "threads", "opfs"],
  "licenses": [
    { "component": "language_model",  "spdx": "Apache-2.0" },
    { "component": "embedding_model", "spdx": "MIT" },
    { "component": "corpus",          "spdx": "CC-BY-4.0" },
    { "component": "runtime",         "spdx": "MIT" },
    { "component": "ui",              "spdx": "MIT" }
  ]
}
```

## Appendix C — Reference layout for the example ELF (informative)

A minimal `.elf` derived from the `wasm-knowledge-chatbot-rs` interface, stripped of WebLLM-specific configuration and configured for wllama/GGUF:

```
hello.elf (ZIP)
├── manifest.json
├── index.html                    # chat shell, input box, scrollback
├── runtime/
│   ├── main.js                   # implements window.elf per §6.3
│   ├── wllama/                   # wllama runtime binding
│   │   ├── single-thread/wllama.wasm
│   │   └── multi-thread/wllama.wasm
│   └── templates/chatml.js       # chat template implementations
├── models/
│   ├── llm/tinyllama-1.1b-q4_k_m.gguf
│   └── embed/all-minilm-l6-v2-q4_k.gguf
├── corpus/
│   ├── chunks.jsonl              # {id, text, source} per line
│   ├── vectors.bin               # 42 × 384 × float32 = 64,512 bytes
│   └── documents/
│       └── handbook.md
├── assets/
│   └── favicon.svg
└── LICENSES/
    ├── apache-2.0.txt
    ├── mit.txt
    └── cc-by-4.0.txt
```
