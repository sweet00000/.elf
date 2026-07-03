# Specification Amendments (drop-in text)

Exact replacement text for the legal/governance gaps in
`ELF-SPEC-v0.2.md` and companions. Since v0.2 is nominally published, the
cleanest path is to land these in a v0.3 draft; applying them to v0.2 with a
change-log note is acceptable while the spec has no external implementers.

---

## A. Document header (all spec documents)

Replace:

> **License of this document:** CC BY NC SA 4.0

With:

> **License of this document:** CC BY 4.0 (see `LICENSE-DOCS.md`).
> Schemas, examples, and test material extracted from this document are
> additionally available under Apache-2.0 (see `LICENSE`).
> **Patent commitment:** see `PATENTS.md`.

Also standardize the name line in every document to:

> **ELF — Embedded Language Format**

with a one-line disambiguation footnote on first use:

> *ELF here is unrelated to the Executable and Linkable Format used for
> compiled binaries on Unix-like systems.*

---

## B. Replace §4.12 `licenses` in full

### 4.12 `licenses`

Licensing declarations make an artifact's redistribution terms inspectable
without unpacking it. They are the mechanism by which an artifact carries
the attribution and license obligations of its bundled content.

An artifact MUST declare a license entry for every resource with role
`model`, `binding-runtime`, or `binding-ui`, and for every canonical
knowledge resource (`document`, `segment-set`) whose content the artifact
author does not own outright. An artifact SHOULD declare a license entry
for every other resource. Derived artifacts (`canonical_content: false`)
inherit the license of their `derived_from` sources unless a separate entry
is declared.

A license entry has the following shape:

```json
{
  "scope": "resource",
  "target": "res:doc.handbook",
  "spdx": "CC-BY-SA-4.0",
  "text_resource": "res:license.ccbysa4",
  "redistributable": true,
  "attribution": {
    "title": "Example Handbook",
    "creator": "Example Authors",
    "source_url": "https://example.com/handbook",
    "changes": "Segmented and reformatted for retrieval."
  }
}
```

| Field | Type | Req. | Description |
|---|---|---|---|
| `scope` | string | **R** | `resource` or `artifact`. |
| `target` | string | C | Resource ID. REQUIRED when `scope` is `resource`. |
| `spdx` | string | **R** | Valid SPDX identifier or SPDX license expression, or `LicenseRef-<tag>` for custom terms. |
| `text_resource` | string | C | Resource ID of the full license text. REQUIRED when `spdx` is a `LicenseRef-` value. |
| `redistributable` | boolean | **R** | Whether the artifact author asserts the resource may be redistributed as packaged. |
| `attribution` | object | C | REQUIRED when the license imposes attribution obligations (for example CC BY, CC BY-SA). |

Attribution fields: `title`, `creator`, `source_url`, `changes` — all
strings, all OPTIONAL individually, but at least one of `creator` or
`source_url` MUST be present when `attribution` is required.

Rules:

1. A resource acquired via `fetch_urls` MUST have a license entry.
2. A license entry with `redistributable: false` MUST NOT be combined with
   packaging the resource bytes in a distributed container; such resources
   MUST be `fetch_urls`-only.
3. Executors MUST make declared licenses and attributions available to the
   user through the trust surface of Section 7.2 without requiring
   developer tools.
4. License declarations are assertions by the artifact author. Executors
   are not required to validate them, and their presence creates no
   warranty by the executor.

---

## C. Add to §4.14 `signatures` (append after existing rules)

5. Signatures under this section are integrity and attribution mechanisms.
   They are not represented to be qualified or legal electronic signatures
   under any regime (for example eIDAS or the U.S. E-SIGN Act), and
   conforming executors MUST NOT present them as such.
6. A signer SHOULD publish a revocation mechanism reachable from
   `identity_hint`. Executors that support a revocation mechanism MUST
   treat a revoked key's signatures as unverified, not as invalid
   artifacts: the artifact remains loadable, but the signature MUST NOT be
   presented as valid.

---

## D. Replace §9.3 `Registries` in full

### 9.3 Registries

The following value spaces are registry-managed: resource roles, operation
names, fulfillment kinds, requirement tokens, permission tokens, binding
names, and template format names.

The authoritative registries are the JSON files in the `/registries`
directory of the specification repository. The registration process,
approval authority, and stability rules are defined in `GOVERNANCE.md`.
Registry additions are additive and do not require a specification version
bump; removals or semantic changes are backward-incompatible and require a
major version change.

Non-standard values MUST use `x-...` or URN-style namespacing and require
no registration.

---

## E. Add to §4.13 `provenance` (append)

To ease downstream compliance documentation (software bills of materials,
model documentation), `provenance` MAY additionally include:

```json
"model_documentation": [
  {
    "resource": "res:model.llm",
    "card_url": "https://example.com/model-card",
    "provider": "Example Lab",
    "training_data_summary_url": "https://example.com/data-summary"
  }
],
"sbom_export": "spdx-2.3"
```

`sbom_export`, when present, names a registered mapping under which the
resource table can be mechanically exported as a software bill of
materials. Tooling SHOULD support at least one such mapping.

---

## F. Change log block (add near top of the amended spec)

## Change log

- **v0.2 → v0.3 (draft):** relicensed documents to CC BY 4.0 with
  Apache-2.0 for extractable material; added patent commitment; restored
  mandatory licensing for redistribution-relevant resources with
  `redistributable` and `attribution` fields (§4.12); clarified legal
  status and revocation handling for signatures (§4.14); defined registry
  governance (§9.3, GOVERNANCE.md); added optional model documentation and
  SBOM export hooks to provenance (§4.13); standardized the format name as
  Embedded Language Format.
