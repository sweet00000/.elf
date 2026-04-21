# ELF v0.2 Conformance Checklist

This checklist complements [ELF-MANIFEST-SCHEMA-v0.2.json](/c:/Users/Sweet/Documents/.ai/.elf/.elf/v0.2/ELF-MANIFEST-SCHEMA-v0.2.json). The schema covers structure. This checklist covers the semantic rules that JSON Schema alone does not fully enforce.

## 1. Manifest and identity

- The manifest validates against `ELF-MANIFEST-SCHEMA-v0.2.json`.
- `elf_version` is exactly `0.2`.
- The artifact digest is computed by canonicalizing the manifest with `signatures` replaced by `[]`, then hashing with SHA-256.
- Any declared signature references that exact artifact digest.
- Unknown critical extensions cause a hard failure.

## 2. Resource graph

- Every referenced resource ID exists exactly once in `resources`.
- Every used resource verifies by declared `sha256` and `size` before use.
- Every behavior-affecting byte is represented in `resources`.
- Any shipped worker script, runtime script, binding adapter, prompt template, tokenizer, index, or UI asset that affects behavior is marked `behavioral: true`.
- Every resource fetched via `fetch_urls` uses HTTPS and verifies before parse or execution.
- Any derived artifact that declares `derived_from` is not presented as canonical content.

## 3. Knowledge model

- If the artifact claims document-backed knowledge, it declares `documents`, `segments`, or both.
- `documents` and `segments` point to canonical content resources.
- `embeddings` and `indices` point to non-canonical derived resources.
- Retrieval results reference stable segment IDs from the declared segment set.

## 4. Interaction and fulfillments

- Every operation listed in `interaction.operations` has a same-named property in `fulfillments`.
- Every fulfillment references only declared resources or declared host capabilities.
- If `selection_policy` is `pinned`, the executor does not silently substitute a different fulfillment.
- Host-capability substitution, when used, is disclosed to the user or developer.
- Scores returned by retrieval are treated as fulfillment-local, not globally comparable.

## 5. Requirements and permissions

- Unmet `requirements` cause refusal to execute.
- Undeclared permissions are not granted.
- If `permissions.network.mode` is `allow-listed`, every allowed origin is explicitly declared.
- If the artifact depends on browser worker availability, it declares `binding.browser-worker` or `binding.browser-module-worker` as appropriate.
- Worker support is treated as a binding requirement, not as a permission.

## 6. Packaging and conformance class

- `elf-container` packages include `manifest.json` at the root and package any standalone-required resources at their declared `path`.
- `elf-single` packages include the manifest plus inert packaged resources with resource IDs and integrity metadata.
- Packaging round-trips preserve artifact digest, resource IDs, and resource hashes.
- `standalone` artifacts use `permissions.network.mode: none` and package all resources required by the selected default fulfillments.
- `network-attached` artifacts declare network permission, allow-list all origins, and integrity-pin all remote bytes.

## 7. Executor behavior

- The executor surfaces artifact title, artifact digest, signature state, requested permissions, granted permissions, and selected fulfillments.
- The executor never executes undeclared or unverified bytes.
- The executor never silently accepts unknown critical extensions.
- The executor makes trust chrome visually distinct from artifact-provided UI in bindings that render a viewer shell.
