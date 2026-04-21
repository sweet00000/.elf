# ELF Format Specification, version 0.2

**ELF - Executable Language Format**  
*Signed, content-addressed, permission-declared language artifacts.*

**Status:** Draft  
**Editors:** Sweet et al.  
**License of this document:** CC BY NC SA 4.0  
**Date:** 2026-04-21

---

## 0. Introduction (non-normative)

An **ELF artifact** is a signed, content-addressed, permission-declared, locally executable language artifact with declarative interaction semantics and optional platform bindings.

An ELF is not defined by HTML, JavaScript, a browser sandbox, a specific model format, or a particular chat UI. Those are binding and distribution choices layered on top of a smaller core. The core defines:

- what the artifact is;
- which bytes affect its behavior;
- what permissions it requests;
- what interaction it declares;
- how its identity is computed;
- how a viewer proves those claims to a user.

This structure is intended to keep ELF stable across changing runtimes. A future executor MAY use a different renderer, accelerator, or local model provider while preserving the same artifact identity, resource graph, permissions, and interaction contract.

### 0.1 Goals

- **Durable identity.** The same logical artifact has the same identity regardless of packaging profile or executor.
- **Hash-complete behavior.** Every behavior-affecting byte is declared and verified.
- **Platform neutrality.** The core artifact is not defined in terms of any one host platform.
- **Local-first execution.** A standalone ELF runs locally after load without further network dependency.
- **Explicit permissions.** Trust-boundary crossings are declared up front and default to deny.
- **Inspectable knowledge.** Canonical source content is distinguishable from derived acceleration artifacts.
- **Substitutable fulfillment.** Executors MAY satisfy declared operations through compatible local capabilities when allowed by the artifact.
- **Fail-closed extensibility.** Unknown critical extensions abort; they do not silently degrade.

### 0.2 Non-goals

- Mandating a single user interface.
- Mandating a single inference runtime, tokenizer, or weight format.
- Concealing artifact contents from a user who wants to inspect them.
- Guaranteeing identical output across different fulfillments unless the artifact pins one fulfillment.

### 0.3 Document conventions

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY**, and **REQUIRED** in this document are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

---

## 1. Layers and scope

ELF v0.2 is layered:

1. **Core artifact model.** Manifest, resources, permissions, interaction contract, identity, signatures.
2. **Fulfillment layer.** How declared operations are satisfied by embedded resources, local host capabilities, or network-attached resources.
3. **Packaging layer.** How the logical artifact is distributed in one file or archive.
4. **Platform binding layer.** How a host exposes the artifact on a concrete platform.

Sections 2 through 7 are normative for every ELF artifact.  
Section 8 is normative only when a packaging profile is claimed.  
Section 9 defines versioning and extension rules.  
Section 10 defines conformance classes.

Companion documents define the browser security profile and the reference browser viewer. Those documents are not the core format.

---

## 2. Terminology

- **Artifact.** The logical ELF described by a manifest plus its declared resource graph.
- **Manifest.** The canonical JSON description of an ELF artifact.
- **Resource.** A named, hashed byte sequence declared by the manifest.
- **Resource ID.** A stable string key that identifies a resource inside the artifact.
- **Behavioral resource.** A resource whose bytes can change execution semantics or user-visible behavior.
- **Canonical content.** Source material intended to remain meaningful independent of the current executor, such as documents, segment records, licenses, or prompt programs.
- **Derived artifact.** A resource produced from other resources for performance or compatibility, such as embeddings, ANN indices, or converted weights.
- **Fulfillment.** A declared way to satisfy a registered operation.
- **Requirement.** A property the executor must have; not a user grant.
- **Permission.** A trust-boundary crossing that must be declared and is subject to user mediation.
- **Binding.** A platform-specific projection of the artifact, such as a browser JavaScript API.
- **Critical extension.** An extension whose unknown semantics would make execution unsafe or incorrect.
- **Artifact digest.** The canonical identity hash of the artifact as defined in Section 6.2.

---

## 3. Core invariants

An ELF artifact is conforming only if all of the following hold:

1. Every byte that can change execution semantics or user-visible behavior is declared in the resource table.
2. Every declared resource has a stable `id`, `media_type`, `size`, and `sha256`.
3. Every declared resource that is used is hash-verified before use.
4. Permissions default to deny when omitted.
5. Unknown critical extensions cause a hard failure.
6. Canonical knowledge resources are distinguishable from derived acceleration resources.
7. Signatures, when present, sign the full artifact digest rather than an ad hoc field list.

---

## 4. The manifest

### 4.1 Format

The manifest is a UTF-8 JSON document. It MUST parse as a valid JSON object. Field ordering is not significant. Canonicalization for identity and signing uses RFC 8785 (JSON Canonicalization Scheme).

Unknown fields MUST be preserved by tooling. Unknown fields MAY be ignored by viewers unless they are introduced by a critical extension declared in `extensions`.

### 4.2 Top-level schema

```json
{
  "elf_version": "0.2",
  "id": "urn:uuid:550e8400-e29b-41d4-a716-446655440000",
  "title": "Example Knowledge Base",
  "description": "A signed language artifact.",
  "created": "2026-04-21T00:00:00Z",
  "locales": ["en"],

  "resources": [ ... ],
  "knowledge": { ... },
  "interaction": { ... },
  "fulfillments": { ... },
  "bindings": [ ... ],

  "requirements": [ ... ],
  "permissions": { ... },
  "extensions": [ ... ],

  "licenses": [ ... ],
  "provenance": { ... },
  "signatures": [ ... ]
}
```

### 4.3 Core fields

| Field | Type | Req. | Description |
|---|---|---|---|
| `elf_version` | string | **R** | Spec version. `"0.2"` for this document. |
| `id` | string | **R** | Stable artifact identifier. SHOULD be a URN. |
| `title` | string | **R** | Human-readable title. |
| `description` | string | O | Short description. |
| `created` | string | **R** | ISO 8601 timestamp of build. |
| `locales` | array | O | BCP 47 tags describing supported human languages. |
| `resources` | array | **R** | Universal resource table. |
| `knowledge` | object | O | Canonical content and derived retrieval artifacts. |
| `interaction` | object | **R** | Declarative interaction contract. |
| `fulfillments` | object | **R** | Ways to satisfy declared operations. |
| `bindings` | array | O | Optional platform bindings. |
| `requirements` | array | O | Required executor capabilities. |
| `permissions` | object | O | Declared trust-boundary crossings. Defaults deny. |
| `extensions` | array | O | Extension declarations. |
| `licenses` | array | O | Licensing information. |
| `provenance` | object | O | Build and source provenance. |
| `signatures` | array | O | Signatures over the artifact digest. |

### 4.4 `resources`

`resources` is the canonical, package-independent resource table. Every resource entry has the following shape:

```json
{
  "id": "res:doc.handbook",
  "media_type": "text/markdown",
  "size": 18342,
  "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "role": "document",
  "behavioral": false,
  "canonical_content": true,
  "path": "documents/handbook.md"
}
```

| Field | Type | Req. | Description |
|---|---|---|---|
| `id` | string | **R** | Unique resource identifier within the artifact. |
| `media_type` | string | **R** | Media type of the raw bytes. |
| `size` | integer | **R** | Raw byte length. |
| `sha256` | string | **R** | Lowercase hex SHA-256 of the raw bytes. |
| `role` | string | **R** | Semantic role. Registered values below. |
| `behavioral` | boolean | **R** | True if the bytes can change execution semantics or user-visible behavior. |
| `canonical_content` | boolean | **R** | True if the bytes are durable source content rather than a derived accelerator or adapter artifact. |
| `path` | string | O | Reference packaging path. |
| `derived_from` | array | O | Resource IDs this resource was derived from. REQUIRED for derived artifacts. |
| `fetch_urls` | array | O | Optional HTTPS URLs from which the exact bytes MAY be fetched if not packaged. |

Registered `role` values in v0.2:

- `document`
- `segment-set`
- `embedding-set`
- `index`
- `model`
- `template`
- `binding-ui`
- `binding-runtime`
- `asset`
- `license-text`
- `provenance`
- `x-...`

Resource rules:

1. Every referenced resource ID MUST exist exactly once in `resources`.
2. Every behavioral byte MUST be represented by a resource with `behavioral: true`.
3. A resource with `fetch_urls` MUST also declare `sha256` and `size`.
4. `fetch_urls` entries MUST use HTTPS.
5. A derived artifact MUST set `canonical_content: false` and declare `derived_from`.
6. Paths are packaging hints, not identity. The resource table is authoritative.

Examples of resources that MUST be declared as behavioral when present:

- model weights;
- tokenizer files;
- prompt templates;
- retrieval indices if the executor may use them for result selection;
- UI HTML, CSS, fonts, images, and icons that affect user-visible output;
- runtime code, worker scripts, and binding adapters.

### 4.5 `knowledge`

`knowledge` distinguishes durable source content from derived retrieval artifacts.

```json
"knowledge": {
  "documents": ["res:doc.handbook"],
  "segments": {
    "resource": "res:segments.main",
    "format": "elf-segments-jsonl/v1"
  },
  "embeddings": [
    {
      "resource": "res:embed.main",
      "dimensions": 384,
      "dtype": "float32"
    }
  ],
  "indices": [
    {
      "resource": "res:index.hnsw",
      "kind": "hnsw"
    }
  ]
}
```

Rules:

1. If an artifact claims document-backed knowledge, it MUST declare `documents`, `segments`, or both.
2. `documents` and `segments` are canonical knowledge resources. They MUST use `canonical_content: true`.
3. `embeddings` and `indices` are derived artifacts. Their referenced resources MUST use `canonical_content: false` and MUST declare `derived_from`.
4. Retrieval results MUST reference stable segment identifiers from the declared segment set.

`elf-segments-jsonl/v1` is a JSON Lines resource. Each line MUST contain at least:

```json
{
  "id": "seg-0001",
  "document": "res:doc.handbook",
  "text": "Chunk text"
}
```

It MAY additionally include:

```json
{
  "start": 1200,
  "end": 1548,
  "anchors": ["section-2.1"],
  "metadata": { "page": 4 }
}
```

### 4.6 `interaction`

`interaction` is the platform-neutral contract the artifact declares for human-facing language behavior.

```json
"interaction": {
  "kind": "chat",
  "operations": ["generate-text", "retrieve-segments"],
  "generate-text": {
    "input": "messages",
    "roles": ["system", "user", "assistant", "tool"],
    "template": {
      "resource": "res:template.chat",
      "format": "elf-template/v1"
    },
    "default_stop": [{ "eos": true }]
  },
  "retrieve-segments": {
    "segments": "res:segments.main",
    "default_k": 5,
    "citation_mode": "segment-id"
  }
}
```

`kind` is a hint, not a restriction. Registered values include `chat`, `search`, `qa`, `summarize`, `translate`, `agent`, and `blank`.

Registered operations in v0.2:

- `generate-text`
- `embed-text`
- `retrieve-segments`
- `read-resource`

The artifact MUST NOT require a platform-specific API name as part of its interaction contract. Platform bindings MAY project the contract into local APIs.

### 4.7 `fulfillments`

`fulfillments` declares how each registered operation can be satisfied.

```json
"fulfillments": {
  "generate-text": [
    {
      "id": "gen.embedded.gguf",
      "kind": "embedded-model",
      "resource": "res:model.llm",
      "interface": "decoder-text/v1",
      "format": "gguf",
      "context_window": 4096,
      "selection_policy": "prefer-embedded"
    },
    {
      "id": "gen.host.local",
      "kind": "host-capability",
      "selector": "urn:elf:capability:text-generation:decoder:v1",
      "minimum_context_window": 4096
    }
  ],
  "retrieve-segments": [
    {
      "id": "retrieve.main",
      "kind": "embedded-index",
      "segments": "res:segments.main",
      "embeddings": "res:embed.main",
      "index": "res:index.hnsw",
      "metric": "cosine"
    }
  ]
}
```

Registered fulfillment `kind` values in v0.2:

- `embedded-model`
- `embedded-index`
- `host-capability`
- `network-resource`
- `x-...`

Fulfillment rules:

1. A fulfillment MUST only reference resources present in the resource table.
2. For every operation named in `interaction.operations`, `fulfillments` MUST contain a property with that exact operation name and at least one compatible fulfillment.
3. An artifact MAY declare more than one compatible fulfillment for an operation.
4. Registered `selection_policy` values are `prefer-embedded`, `prefer-host`, and `pinned`.
5. An executor MAY choose among compatible fulfillments unless the artifact marks one as `selection_policy: "pinned"`.
6. `host-capability` allows the executor to satisfy the operation through a compatible local capability rather than an embedded model.
7. `network-resource` is not valid for standalone conformance.
8. If an artifact declares both embedded and host-provided fulfillments, the artifact remains the same logical ELF. Only the selected fulfillment changes.

### 4.8 `bindings`

Bindings are optional projections of the core artifact into specific execution environments.

```json
"bindings": [
  {
    "name": "browser-js",
    "version": "0.2",
    "requirements": ["binding.browser-dom", "binding.browser-esm"],
    "entry": {
      "ui": "res:binding.browser.html",
      "runtime": "res:binding.browser.js"
    }
  }
]
```

Binding rules:

1. Bindings are optional. The absence of a binding does not invalidate the core artifact.
2. A binding entry resource MUST be a declared behavioral resource.
3. Platform bindings MUST NOT redefine artifact identity, permissions, or signatures.
4. Bindings MAY expose platform-native APIs, but those APIs are not themselves the ELF core contract.

### 4.9 `requirements`

`requirements` declares properties the executor must possess. Requirements are not user grants.

Registered requirement tokens in v0.2:

- `compute.local`
- `compute.simd`
- `compute.shared-memory`
- `acceleration.gpu`
- `storage.persistent`
- `binding.browser-dom`
- `binding.browser-esm`
- `binding.browser-worker`
- `binding.browser-module-worker`

Unmet requirements MUST cause the executor to refuse execution with a clear user-facing error.

Web Workers are not a core ELF concept. In v0.2 they are a browser-binding execution detail. If an artifact's browser binding requires worker availability, it SHOULD declare `binding.browser-worker`, and it SHOULD declare `binding.browser-module-worker` when module workers are required. Worker scripts shipped inside the artifact MUST be declared as behavioral resources.

### 4.10 `permissions`

`permissions` declares trust-boundary crossings. When omitted, every permission defaults to deny.

```json
"permissions": {
  "network": { "mode": "none" },
  "filesystem": { "read": false, "write": false },
  "clipboard": { "read": false, "write": false },
  "devices": { "microphone": false, "camera": false },
  "automation": false
}
```

Rules:

1. Permissions MUST be explicit and user-visible.
2. `network.mode` is `none` or `allow-listed`.
3. If `network.mode` is `allow-listed`, `origins` MUST be present and MUST match the origins implied by any `fetch_urls`.
4. Declaring a requirement does not imply the corresponding permission.
5. Executors MUST NOT grant undeclared permissions.

### 4.11 `extensions`

`extensions` is an array of declarations:

```json
"extensions": [
  { "name": "x-example-foo", "version": "1.0", "critical": false }
]
```

Rules:

1. Extension-defined field names MUST be namespaced with `x-`.
2. Unknown non-critical extensions MAY be ignored.
3. Unknown critical extensions MUST cause execution to abort.

### 4.12 `licenses`

Each significant logical component present in the artifact SHOULD declare a license. At minimum, artifacts SHOULD declare licensing for:

- canonical documents or corpus content;
- embedded models;
- runtime and binding code;
- bundled assets when separately licensed.

A license entry MAY point to a resource ID carrying the full text:

```json
{
  "scope": "resource",
  "target": "res:model.llm",
  "spdx": "Apache-2.0",
  "text_resource": "res:license.apache2"
}
```

### 4.13 `provenance`

`provenance` is informational but strongly recommended:

```json
"provenance": {
  "builder": "elf-build/0.2.0",
  "built_at": "2026-04-21T00:00:00Z",
  "source_documents": [
    {
      "resource": "res:doc.handbook",
      "origin_url": "https://example.com/handbook.md",
      "retrieved_at": "2026-04-20T23:00:00Z"
    }
  ],
  "reproducible_seed": null
}
```

### 4.14 `signatures`

Signatures in v0.2 sign the full artifact digest, not a subset of fields.

```json
"signatures": [
  {
    "algorithm": "ed25519",
    "artifact_digest": "sha256:abc123...",
    "public_key": "<base64>",
    "signature": "<base64>",
    "identity_hint": "did:web:example.com"
  }
]
```

Rules:

1. A signature MUST reference the artifact digest defined in Section 6.2.
2. Executors MUST verify declared signatures before presenting them as valid.
3. Failed verification MUST be surfaced clearly.
4. Unsigned ELF artifacts are valid unless a higher-level policy forbids them.

---

## 5. Declarative interaction semantics

### 5.1 Registered operation semantics

This section defines the platform-neutral request and response shapes for the registered operations.

#### `generate-text`

Request:

```json
{
  "messages": [
    { "role": "system", "content": "You are a guide." },
    { "role": "user", "content": "Summarize the handbook." }
  ],
  "max_output_tokens": 512,
  "temperature": 0.7,
  "top_p": 0.95,
  "seed": 1234,
  "stop": [{ "eos": true }]
}
```

Streaming response chunks:

```json
{ "delta": "Hello", "done": false }
{ "delta": "", "done": true, "stop_reason": "eos" }
```

#### `embed-text`

Request:

```json
{
  "texts": ["hello world"]
}
```

Response: an ordered sequence of numeric vectors whose dimensionality matches the selected fulfillment.

#### `retrieve-segments`

Request:

```json
{
  "query": "vacation policy",
  "k": 5
}
```

Response items:

```json
{
  "segment_id": "seg-0007",
  "score": 0.92,
  "document": "res:doc.handbook",
  "text": "Employees accrue...",
  "anchors": ["section-4.2"]
}
```

#### `read-resource`

Request:

```json
{ "resource": "res:doc.handbook" }
```

Response: raw bytes of the addressed resource after hash verification.

### 5.2 `elf-template/v1`

ELF v0.2 introduces a declarative prompt template format intended to replace unrestricted string templating.

An `elf-template/v1` resource is JSON with this shape:

```json
{
  "version": 1,
  "sequence": [
    { "literal": "<|begin|>" },
    {
      "repeat": "messages",
      "sequence": [
        { "literal": "<|role:" },
        { "field": "role" },
        { "literal": "|>\n" },
        { "field": "content" },
        { "literal": "\n" }
      ]
    },
    { "literal": "<|role:assistant|>\n" }
  ],
  "stop": [{ "eos": true }]
}
```

Allowed sequence items in v0.2:

- `{ "literal": string }`
- `{ "field": "role" | "content" | "name" }`
- `{ "repeat": "messages", "sequence": [...] }`

Rules:

1. `elf-template/v1` evaluation MUST be deterministic.
2. Template evaluation MUST NOT execute arbitrary code.
3. Executors MAY support registered legacy named templates for compatibility, but artifacts SHOULD prefer `elf-template/v1`.

### 5.3 Retrieval semantics

When `retrieve-segments` is declared, the executor MUST return only segment IDs from the declared segment set. Approximate indices MAY be used, but the selected fulfillment MUST still produce ranked results over that same segment universe.

Scores are fulfillment-local. Executors MUST NOT imply that scores are comparable across different fulfillments unless explicitly documented.

---

## 6. Identity, integrity, and caching

### 6.1 Resource verification

Every declared resource used by the executor MUST be hash-verified against its manifest `sha256` before use. A mismatch is a fatal error.

For resources obtained via `fetch_urls`, the fetched bytes MUST verify against the declared `sha256` and `size` before the bytes are executed, parsed, or handed to a fulfillment.

### 6.2 Artifact digest

The canonical identity of an ELF artifact is the SHA-256 of the manifest in canonical form, after replacing `signatures` with an empty array.

Algorithm:

1. Parse the manifest.
2. Replace `signatures` with `[]`. If absent, behave as if it were present and empty.
3. Canonicalize the resulting JSON with RFC 8785.
4. Compute SHA-256 over the resulting UTF-8 bytes.

Executors SHOULD display this as the artifact fingerprint to users, prefixed `sha256:`.

### 6.3 Resource tree root

Executors SHOULD also compute a resource tree root for diagnostics and caching:

1. Sort resources by `id` lexicographically.
2. For each resource, compute a leaf digest over `id`, `media_type`, `size`, and `sha256`.
3. Concatenate the leaf digests in order.
4. SHA-256 the concatenation.

The resource tree root does not replace the artifact digest. It is a supplementary integrity summary.

### 6.4 Cache keying

Shared content caches SHOULD be keyed by resource `sha256`.

Derived caches SHOULD be keyed by:

- artifact digest;
- derivation algorithm identifier;
- digests of the source resources named in `derived_from`.

Executors MUST provide a way to inspect and clear persistent caches.

---

## 7. Security model

### 7.1 Core security invariants

A conforming executor MUST enforce all of the following:

- no use of undeclared bytes;
- no use of unverified bytes;
- no granting of undeclared permissions;
- no silent acceptance of unknown critical extensions;
- no hidden substitution of a different artifact identity.

### 7.2 User-visible trust signals

Executors MUST surface, without requiring developer tools:

- artifact title;
- artifact digest;
- signature status;
- permissions requested;
- permissions actually granted;
- selected fulfillment for each declared operation when relevant;
- whether network access occurred, and to which origins.

### 7.3 Fulfillment transparency

If an executor satisfies an operation through a host capability rather than an embedded model, it MUST disclose that selection to the user or developer on inspection. Executors MUST NOT present host-substituted fulfillment as if the embedded resource was used.

### 7.4 Network-attached execution

An ELF artifact may be network-attached only if:

1. network permission is explicitly declared;
2. allowed origins are explicitly declared;
3. remote bytes are named in the resource table or fulfillment declaration with exact integrity;
4. fetched bytes are verified before use.

---

## 8. Reference packaging profiles

The logical ELF artifact is packaging-neutral. This section defines two reference distribution profiles.

### 8.1 `elf-container`

An `elf-container` file is a ZIP archive.

Requirements:

- `manifest.json` MUST exist at the root.
- Every resource with a declared `path` that is required for standalone execution MUST exist at that path.
- Paths MUST use forward slashes.
- Paths MUST NOT contain `..`, absolute prefixes, or symbolic links.
- Unknown archive members MAY be present and MUST be ignored unless referenced by the manifest.

### 8.2 `elf-single`

An `elf-single` file is a single HTML5 document intended for the browser binding.

Requirements:

1. It MUST contain the manifest in a `<script type="application/elf-manifest">` element.
2. Each packaged resource MUST appear in an inert element carrying at minimum:
   - resource ID;
   - path if any;
   - media type;
   - size;
   - SHA-256.
3. It MUST only be used for artifacts that declare a browser binding.
4. It MUST NOT initiate undeclared network requests.

### 8.3 Equivalence

Two packaged files are equivalent if they describe the same logical artifact digest and their declared resources verify.

A packaging round trip MUST preserve:

- manifest semantics;
- artifact digest;
- resource IDs;
- resource hashes.

---

## 9. Versioning and extensibility

### 9.1 Versioning

This document defines `elf_version: "0.2"`. Backward-incompatible changes bump the major version. Additive changes bump the minor version.

Executors MUST refuse artifacts with a major version they do not implement.

### 9.2 Namespaced extensions

Any extension-defined field name MUST begin with `x-`. Extension semantics that affect correctness, security, or user-visible meaning SHOULD be declared in `extensions` with `critical: true`.

### 9.3 Registries

The following are registry-managed in v0.2:

- resource roles;
- operation names;
- fulfillment kinds;
- requirement tokens;
- permission tokens;
- binding names;
- template format names.

Non-standard values MUST use `x-...` or URN-style namespacing.

---

## 10. Conformance

### 10.1 Core artifact conformance

An ELF artifact is conforming at version V if and only if:

- its manifest validates against Section 4;
- every referenced resource exists in the resource table;
- every used resource verifies by hash;
- declared operations, fulfillment kinds, bindings, and template formats are registered at version V or correctly namespaced;
- unknown critical extensions are rejected;
- it passes all applicable tests in the version V conformance suite.

### 10.2 Standalone conformance

A conforming ELF has the **standalone** class if and only if:

- `permissions.network.mode` is `none`;
- every resource required by the selected default fulfillments is packaged locally;
- the artifact can execute after load without additional network access.

### 10.3 Network-attached conformance

A conforming ELF has the **network-attached** class if and only if:

- network permission is explicitly declared;
- every remote byte is integrity-pinned;
- all network origins are allow-listed.

### 10.4 Executor conformance

An executor is conforming at version V if and only if:

- it enforces Sections 6 and 7;
- it does not silently grant undeclared permissions;
- it fails closed on unknown critical extensions;
- it surfaces the trust signals of Section 7.2;
- it correctly implements the operations, bindings, and template formats it advertises;
- it passes all applicable tests in the version V conformance suite.

---

## Appendix A - Example minimal manifest (informative)

```json
{
  "elf_version": "0.2",
  "id": "urn:uuid:00000000-0000-4000-8000-000000000002",
  "title": "Hello ELF v0.2",
  "created": "2026-04-21T00:00:00Z",
  "resources": [
    {
      "id": "res:doc.handbook",
      "media_type": "text/markdown",
      "size": 1024,
      "sha256": "0000000000000000000000000000000000000000000000000000000000000000",
      "role": "document",
      "behavioral": false,
      "canonical_content": true,
      "path": "documents/handbook.md"
    },
    {
      "id": "res:segments.main",
      "media_type": "application/x-ndjson",
      "size": 2048,
      "sha256": "1111111111111111111111111111111111111111111111111111111111111111",
      "role": "segment-set",
      "behavioral": false,
      "canonical_content": true,
      "path": "corpus/segments.jsonl"
    },
    {
      "id": "res:template.chat",
      "media_type": "application/vnd.elf.template+json",
      "size": 256,
      "sha256": "2222222222222222222222222222222222222222222222222222222222222222",
      "role": "template",
      "behavioral": true,
      "canonical_content": true,
      "path": "interaction/chat-template.json"
    },
    {
      "id": "res:model.llm",
      "media_type": "application/octet-stream",
      "size": 734003200,
      "sha256": "3333333333333333333333333333333333333333333333333333333333333333",
      "role": "model",
      "behavioral": true,
      "canonical_content": false,
      "path": "models/llm/model.gguf"
    }
  ],
  "knowledge": {
    "documents": ["res:doc.handbook"],
    "segments": {
      "resource": "res:segments.main",
      "format": "elf-segments-jsonl/v1"
    }
  },
  "interaction": {
    "kind": "chat",
    "operations": ["generate-text"],
    "generate-text": {
      "input": "messages",
      "roles": ["system", "user", "assistant"],
      "template": {
        "resource": "res:template.chat",
        "format": "elf-template/v1"
      },
      "default_stop": [{ "eos": true }]
    }
  },
  "fulfillments": {
    "generate-text": [
      {
        "id": "gen.embedded.gguf",
        "kind": "embedded-model",
        "resource": "res:model.llm",
        "interface": "decoder-text/v1",
        "format": "gguf",
        "context_window": 4096,
        "selection_policy": "pinned"
      }
    ]
  },
  "requirements": ["compute.local"],
  "permissions": {
    "network": { "mode": "none" }
  }
}
```
