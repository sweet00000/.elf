# ELF Security and Cache Model, version 0.2

**Companion to ELF Format Specification v0.2**  
**Status:** Design draft  
**Date:** 2026-04-21

---

## 0. Purpose

This document makes the v0.2 security posture concrete. It has two jobs:

1. define the security invariants expected of any conforming executor; and
2. define a generic cache model that works for content-addressed resources, derived artifacts, and optional network-attached fulfillments.

This is a companion to the core format. It does not redefine artifact identity or the manifest model.

---

## Part I - Security model

### 1. Threat model

The v0.2 model assumes four adversaries:

**A1. Malicious artifact publisher.** Publishes an ELF that tries to:

- execute undeclared code;
- substitute undeclared bytes at runtime;
- request more permission than the user expects;
- blur the distinction between canonical knowledge and derived outputs;
- spoof identity or signature state.

**A2. Malicious host application or embedding page.** Attempts to:

- tamper with the artifact before execution;
- read private interaction state;
- falsify trust signals shown to the user;
- force a different fulfillment than the user expects.

**A3. Malicious network source.** For network-attached resources:

- substitutes bytes at a declared URL;
- serves stale or hostile content;
- observes request patterns.

**A4. Malicious fulfillment provider.** A host-provided local capability or plugin attempts to:

- claim compatibility while violating the declared interaction semantics;
- misreport which model or index was used;
- overreach beyond declared permissions.

Non-goals:

- DRM;
- preventing inspection by a user who wants to extract bytes;
- eliminating all side channels.

### 2. Security invariants

Every conforming executor MUST enforce these invariants:

1. **Hash-complete execution.** No byte influences behavior unless it is declared in the resource table or explicitly declared as a network-attached fulfillment input with integrity.
2. **Permission default deny.** Omitted permissions are denied.
3. **Critical fail-closed behavior.** Unknown critical extensions abort.
4. **Stable identity.** Trust decisions are anchored to the artifact digest, not to packaging names or URLs.
5. **Canonical vs derived separation.** Executors do not present derived artifacts as if they were source content.
6. **Fulfillment transparency.** Executors disclose whether an embedded resource, host capability, or network-attached resource was selected.

### 3. Requirements vs permissions

v0.2 separates executor properties from trust-boundary crossings.

**Requirements** describe what the executor can do:

- local compute;
- SIMD;
- GPU access;
- persistent storage;
- browser DOM support.

**Permissions** describe what the artifact is allowed to cross:

- network;
- filesystem;
- clipboard;
- devices;
- automation.

This distinction is security-critical. `acceleration.gpu` is not the same thing as permission to read files. `binding.browser-dom` is not the same thing as permission to talk to the network.

Executors MUST NOT infer permissions from requirements.

### 4. Resource verification

Every resource used by the executor MUST be verified against its declared hash and size before use.

This applies equally to:

- documents;
- templates;
- models;
- bindings;
- styles and assets;
- retrieval indices.

Verification rules:

1. Hash mismatch is a fatal error.
2. Size mismatch is a fatal error.
3. A resource obtained from `fetch_urls` MUST verify before parse, execution, indexing, or model load.
4. A failed fetch verification MUST NOT silently fall back to unverifiable bytes.

### 5. Network-attached resources

Network-attached execution is allowed only when the manifest explicitly permits it.

Rules:

1. Every fetched byte sequence MUST have a declared resource entry or fulfillment record with exact integrity.
2. Every allowed origin MUST appear in the manifest permission allow-list.
3. Origins MUST use HTTPS.
4. Executors MUST reject redirects to undeclared origins.
5. Executors MUST surface that network access occurred.

The network is a transport, not a source of authority. Authority comes from the artifact digest plus declared resource integrity.

### 6. Signatures and trust presentation

Signatures in v0.2 sign the artifact digest, not a list of chosen fields. This is important because field-list signing is easy to misunderstand and easy to under-specify.

Executors MUST display one of:

- `Verified` plus signer identity;
- `Invalid signature`;
- `Unsigned`.

Executors MUST NOT collapse `invalid` and `unsigned` into the same state.

### 7. Canonical knowledge vs derived artifacts

Executors MUST preserve the distinction between:

- canonical documents and segment records; and
- derived embeddings, ANN indices, converted weights, cached projections, and model outputs.

Derived artifacts MAY accelerate execution. They MUST NOT become the only inspectable representation of source knowledge if the artifact claims document-backed content.

### 8. User-visible trust signals

A conforming executor MUST surface, without requiring developer tools:

- artifact title;
- artifact digest;
- signature state;
- requested permissions;
- granted permissions;
- selected binding, if any;
- selected fulfillment for each declared operation when relevant;
- whether network access occurred and to which origins.

If the executor supports substitution by host capability, the inspection UI MUST indicate that substitution explicitly.

### 9. Incident handling

When an executor detects:

- hash mismatch;
- size mismatch;
- unknown critical extension;
- invalid signature;
- undeclared permission use;
- network request to an undeclared origin;

it MUST:

1. halt the affected load or operation;
2. surface the specific reason;
3. preserve the artifact digest and relevant resource IDs for reporting;
4. avoid silent retry with weaker guarantees.

---

## Part II - Generic cache model

### 10. Why v0.2 needs a broader cache model

v0.1 focused on model deduplication. v0.2 needs a broader cache model because the artifact center is now a resource graph, not only a bundled model.

Executors may want to cache:

- large embedded models;
- fetched resources for network-attached artifacts;
- converted or projected model formats;
- derived retrieval indices;
- parsed segment sets;
- other deterministic derived artifacts.

### 11. Cache classes

v0.2 distinguishes three cache classes.

#### 11.1 Content cache

Stores raw declared resources keyed by resource hash.

Key:

- `sha256` of the resource bytes.

Properties:

- shared across artifacts;
- safe for deduplication;
- valid only after hash verification.

#### 11.2 Artifact metadata cache

Stores small metadata keyed by artifact digest.

Examples:

- title;
- signer identities;
- list of resource IDs;
- previous inspection state.

This cache MUST NOT be treated as authoritative over the artifact bytes themselves.

#### 11.3 Derived cache

Stores deterministic outputs generated from source resources.

Key components:

- artifact digest;
- derivation algorithm identifier;
- digests of source resources named in `derived_from`;
- relevant executor version if the algorithm is executor-specific.

Examples:

- converted model formats;
- tokenizer projections;
- ANN structures;
- normalized vector matrices.

### 12. Cache rules

1. A content cache entry MUST verify by hash on write and SHOULD verify on read.
2. A derived cache entry MUST be invalidated when any source digest changes.
3. A derived cache entry MUST NOT be mistaken for canonical content.
4. Users MUST be able to inspect, evict, and disable persistent caches.
5. Cache presence MUST NOT be the only source of trust. Trust still comes from manifest-declared integrity.

### 13. Privacy and fingerprinting

Content-addressed caches create fingerprinting risk. Mitigations:

- cache scope SHOULD be executor-local rather than shared across unrelated origins or applications;
- users SHOULD be able to clear caches easily;
- artifacts SHOULD NOT be able to probe cache state except through normal load behavior;
- inspection UIs SHOULD explain what is cached and why.

### 14. Network-attached cache flow

For a resource with `fetch_urls`:

1. check content cache by declared hash;
2. on hit, use the cached bytes;
3. on miss, fetch from an allow-listed origin;
4. verify size and hash;
5. store in the content cache by hash;
6. hand verified bytes to the executor.

This means a network-attached artifact still benefits from the same content-addressed cache as a standalone one.

---

## Part III - Browser security profile

This section is specific to the `browser-js` binding. It is not the definition of ELF itself.

### 15. Browser binding architecture

A secure browser executor SHOULD use two isolation boundaries:

1. **Host page -> viewer shell** using a cross-origin iframe.
2. **Viewer shell -> artifact execution context** using a second isolated context inside the viewer.

The purpose of the second boundary is to ensure that artifact-provided UI and runtime code do not run in the viewer's privileged chrome.

### 16. Browser sandbox profile

Recommended outer embed:

```html
<iframe
  src="https://viewer.elf-format.org/embed?src=..."
  sandbox="allow-scripts"
  credentialless
  referrerpolicy="no-referrer"
  allow=""
></iframe>
```

Recommended properties:

- no `allow-same-origin` against the host page;
- no undeclared permission policy grants;
- viewer chrome rendered outside the artifact execution context.

### 17. Browser CSP principles

The viewer shell SHOULD use a CSP that allows its own scripts and workers but not arbitrary artifact network access.

The inner artifact execution context SHOULD use:

- `default-src 'none'`;
- script and worker sources only as required by declared binding resources;
- `connect-src 'none'` unless network permission is declared;
- explicit allow-listed origins only when declared.

The browser CSP is a binding enforcement tool for the manifest permission model. It is not the source of the permission model.

### 18. Host <-> artifact communication

The host page and browser artifact context SHOULD communicate only through a small, schema-validated message surface. Unknown message types SHOULD be ignored, not forwarded.

The message surface MUST NOT provide arbitrary code execution, arbitrary DOM access, or undeclared permission escalation.

### 19. Browser trust chrome

The browser viewer MUST keep its trust chrome visually distinct from artifact-provided UI.

The following belong to viewer chrome, not artifact UI:

- artifact digest;
- signature state;
- permissions granted;
- selected fulfillment;
- terminate button;
- cache controls.

An artifact MAY render anything inside its own UI. That UI is not a trusted source for identity or permission state.
