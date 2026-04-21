# ELF Reference Viewer - Architecture, version 0.2

**Companion to ELF Format Specification v0.2**  
**Status:** Design draft  
**Date:** 2026-04-21

---

## 0. Purpose

This document specifies the architecture of the **ELF Reference Viewer** for v0.2.

The Reference Viewer is one implementation of the `browser-js` binding. It is not the ELF core format. Its job is to:

- load packaged ELF artifacts;
- validate the v0.2 manifest and resource graph;
- surface trust signals;
- select and materialize fulfillments;
- project the core interaction contract into a browser-native API.

The Reference Viewer demonstrates that the v0.2 core can be implemented in the browser without making the browser itself the definition of ELF.

---

## 1. Position in the stack

The v0.2 stack is:

1. **Core artifact** - manifest, resources, operations, permissions, signatures.
2. **Fulfillments** - embedded models, host capabilities, network-attached resources.
3. **Packaging** - `elf-container` ZIP or `elf-single` HTML.
4. **Browser binding** - `browser-js`.
5. **Reference Viewer** - one concrete browser executor of `browser-js`.

The Reference Viewer implements layers 3 through 5. It does not redefine layers 1 and 2.

---

## 2. High-level architecture

```text
Host page
  |
  |  cross-origin iframe
  v
Viewer shell
  |
  +-- Artifact loader
  +-- Manifest validator
  +-- Resource verifier
  +-- Fulfillment selector
  +-- Cache manager
  +-- Trust chrome
  |
  |  isolated execution context
  v
Binding adapter
  |
  +-- browser-js runtime surface
  +-- selected model backend or host capability adapter
  +-- retrieval adapter
  +-- resource reader
```

Two design choices matter most in v0.2:

1. the **artifact loader** operates on the platform-neutral resource graph; and
2. the **binding adapter** is responsible for turning that graph into browser-native behavior.

That split keeps the browser implementation from leaking into the core format.

---

## 3. Main components

### 3.1 Viewer shell

The viewer shell is the privileged browser code that:

- parses viewer URLs and local files;
- loads packaged ELF bytes;
- validates the manifest;
- computes the artifact digest;
- surfaces trust chrome;
- manages persistent caches;
- creates the isolated artifact execution context.

The viewer shell MUST NOT execute artifact-provided runtime or UI code directly in its own context.

### 3.2 Artifact loader

The loader is a pure module that understands:

- `elf-container`;
- `elf-single`;
- the v0.2 manifest shape;
- the resource table;
- artifact digest computation.

Suggested interface:

```typescript
interface LoadedArtifact {
  manifest: Manifest;
  artifactDigest: string;
  profile: 'container' | 'single';
  getResource(id: string): Promise<Blob>;
  listResources(): ResourceRecord[];
}
```

The loader is responsible for:

1. parsing packaging;
2. locating `manifest.json` or the embedded manifest;
3. validating the manifest schema;
4. building a resource index keyed by resource ID;
5. verifying resources lazily on access.

### 3.3 Manifest validator

The validator enforces the core rules:

- required fields present;
- every referenced resource exists;
- required permissions and requirement tokens are well-formed;
- unknown critical extensions fail;
- signatures reference the correct artifact digest.

### 3.4 Resource verifier

The verifier handles:

- size checks;
- SHA-256 verification;
- allow-listed fetching for `fetch_urls`;
- caching of verified content by resource hash.

The verifier MUST treat all behavioral resources the same way. Models are not special. Templates, runtime code, CSS, indices, and UI assets all pass through the same integrity gate.

### 3.5 Fulfillment selector

The selector chooses how to satisfy each declared operation.

Possible choices:

- embedded resource fulfillment;
- host capability fulfillment;
- network-attached resource fulfillment.

Selection inputs:

- artifact manifest;
- executor capabilities;
- user policy;
- cache state.

Selection outputs:

- selected fulfillment ID;
- disclosure state for trust chrome;
- concrete backend configuration for the binding adapter.

### 3.6 Binding adapter

The binding adapter maps the core interaction contract into the browser-native API.

For `browser-js`, the adapter is responsible for:

- creating the isolated artifact execution context;
- projecting core operations into JavaScript methods;
- serving verified resource bytes to artifact-provided binding code;
- routing retrieval and model calls to the selected fulfillment.

When a backend uses Web Workers or module workers, those remain binding implementation details rather than core ELF semantics. If the artifact itself depends on worker availability, it SHOULD declare the corresponding browser binding requirement token, and any shipped worker script MUST appear in the resource table as a behavioral resource.

### 3.7 Cache manager

The viewer maintains three caches, matching the companion cache document:

- content cache keyed by resource hash;
- artifact metadata cache keyed by artifact digest;
- derived cache keyed by artifact digest plus derivation inputs.

### 3.8 Trust chrome

The trust chrome is viewer-owned UI that shows:

- title;
- artifact digest;
- signature state;
- permissions requested and granted;
- selected fulfillments;
- network activity;
- terminate and cache controls.

Trust chrome is not optional. It is the only trustworthy identity surface in the browser binding.

---

## 4. Load pipeline

### 4.1 Open flow

The Reference Viewer follows this sequence:

1. Receive packaged bytes from URL, file picker, drag-and-drop, or host embed.
2. Detect `elf-container` or `elf-single`.
3. Parse the manifest.
4. Replace `signatures` with `[]`, canonicalize, and compute the artifact digest.
5. Validate extensions, requirements, permissions, and resource references.
6. Verify signatures, if present.
7. Build the resource index.
8. Select fulfillments for declared operations.
9. Materialize the browser binding.
10. Expose the browser binding API and mark the artifact ready.

### 4.2 Resource access flow

When any part of the viewer or binding requests a resource:

1. locate the resource record by ID;
2. check the content cache by hash;
3. if absent and `fetch_urls` exist, fetch only from allow-listed origins;
4. otherwise load from the packaged bytes;
5. verify size and hash;
6. cache verified bytes by hash;
7. return a `Blob` or stream.

This pipeline is identical for documents, templates, weights, indices, and binding code.

### 4.3 Failure modes

Load MUST stop on:

- manifest validation failure;
- unknown critical extension;
- unmet requirement;
- undeclared permission need;
- signature mismatch when presenting a verified state;
- resource hash mismatch.

The error UI MUST include the artifact digest.

---

## 5. Browser binding API

The v0.2 browser binding is a projection of the core operations, not the core itself.

The Reference Viewer exposes a browser-native object:

```typescript
interface BrowserElfArtifact extends EventTarget {
  readonly manifest: Manifest;
  readonly artifactDigest: string;
  readonly ready: Promise<void>;

  generateText(req: GenerateTextRequest): AsyncIterable<GenerateTextChunk>;
  embedText(req: EmbedTextRequest): Promise<Float32Array[]>;
  retrieveSegments(req: RetrieveSegmentsRequest): Promise<RetrievedSegment[]>;
  readResource(id: string): Promise<Blob>;
  terminate(): Promise<void>;
}
```

Suggested event names:

- `artifact-ready`
- `resource-progress`
- `fulfillment-selected`
- `error`

Notes:

- This API replaces the older idea that `window.elf` defined the format.
- Another binding on another platform MAY expose the same core operations through a different surface.

---

## 6. Fulfillment handling

### 6.1 Embedded model fulfillment

For `embedded-model`, the viewer:

1. loads the resource by ID;
2. verifies it;
3. optionally consults the derived cache for converted forms;
4. hands verified bytes to the chosen browser backend.

### 6.2 Host capability fulfillment

For `host-capability`, the viewer:

1. checks whether a compatible local capability is available;
2. maps the core request shape to the host capability API;
3. records the substitution in trust chrome and inspection state.

Host capability fulfillment MUST NOT pretend that the embedded model was used.

### 6.3 Network-attached fulfillment

For `network-resource`, the viewer:

1. checks network permission;
2. checks content cache;
3. fetches only from allow-listed origins;
4. verifies bytes;
5. hands verified bytes to the backend.

---

## 7. Retrieval architecture

The Reference Viewer treats retrieval as another fulfillment rather than a hard-coded browser feature.

Possible retrieval paths:

- use packaged segment and embedding resources directly;
- use a packaged ANN index;
- use a derived cached index built from canonical segments and embeddings;
- delegate to a compatible host capability if declared.

Whatever path is chosen, results MUST reference the declared segment IDs from the artifact.

---

## 8. Security architecture

### 8.1 Isolation

The browser executor SHOULD use:

- a viewer shell in one browsing context; and
- an isolated artifact execution context for artifact-provided code.

The artifact context SHOULD NOT have direct access to viewer chrome state, cache management, or elevated browser privileges.

### 8.2 Permission enforcement

The viewer shell enforces manifest permissions:

- network disabled unless declared;
- origins restricted to allow-list when declared;
- no implicit filesystem or device access;
- no undeclared automation surfaces.

### 8.3 Trust surfaces

The viewer MUST visually distinguish:

- viewer-owned trust chrome; and
- artifact-owned UI.

Artifact UI may say anything. Trust chrome is the source of truth for identity, signatures, permissions, and selected fulfillments.

### 8.4 Host embedding

When embedded by another page, the host page SHOULD interact with the viewer through a narrow `postMessage` surface. The host page MUST NOT gain direct access to the artifact DOM or internal state.

---

## 9. `elf-single` handling

In v0.2, `elf-single` is treated as a packaging profile for the browser binding, not as the definition of ELF itself.

The Reference Viewer extracts:

- the embedded manifest;
- inert packaged resources carrying resource IDs and integrity metadata;
- the browser binding entry resources.

The viewer still computes identity from the v0.2 manifest, not from the HTML bytes as such.

---

## 10. Conformance strategy

The Reference Viewer is built alongside a conformance suite covering:

- artifact digest computation;
- resource-table completeness checks;
- signature verification;
- critical extension rejection;
- standalone vs network-attached policy;
- host capability substitution disclosure;
- retrieval result identity by segment ID;
- packaging round-trip stability.

The browser binding conformance suite is a supplement to the core ELF suite, not a replacement for it.
