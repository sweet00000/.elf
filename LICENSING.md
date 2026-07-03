# Licensing

This repository uses different licenses for different kinds of material. The goal
is simple: anyone — including commercial implementers — may implement, extend,
and ship software based on this specification without asking permission.

## What is licensed how

| Material | License | File |
|---|---|---|
| Specification prose (`ELF-SPEC-*.md`, `ELF-SECURITY-AND-CACHE-*.md`, `ELF-REFERENCE-VIEWER-*.md`, conformance checklists) | [CC BY 4.0](LICENSE-DOCS.md) | `LICENSE-DOCS.md` |
| JSON schemas, manifest examples, test vectors, conformance suites | [Apache License 2.0](LICENSE) | `LICENSE` |
| Reference implementation code (`index.html`, viewers, builders, scripts) | [Apache License 2.0](LICENSE) | `LICENSE` |

If a file does not state otherwise, code and machine-readable material is
Apache-2.0 and prose is CC BY 4.0.

## Why the split

- **Schemas and test vectors must be copied verbatim into implementations.**
  Apache-2.0 permits that in any product, commercial or not, and includes an
  express patent grant with defensive termination.
- **Spec prose is meant to be quoted, translated, and excerpted.** CC BY 4.0
  permits that with attribution and no field-of-use restriction.
- Nothing in this repository restricts commercial use.

## Implementing the specification

Implementing the ELF format does not require copying the specification prose.
Field names, data structures, algorithms, and wire formats described here may
be implemented freely. See `PATENTS.md` for the patent commitment covering
conforming implementations.

## Attribution for the prose

When redistributing or adapting the specification documents, attribute as:

> "ELF Format Specification" by the ELF Editors, licensed under CC BY 4.0.
> <https://github.com/sweet00000/.elf>

## Relicensing history

Prior to 2026-07-03, this repository was published under
CC BY-NC-SA 4.0. All material as of that date was authored solely by the
copyright holder, who relicensed it as described above. Versions retrieved
before that date remain available under the earlier license; all current and
future versions are offered under the terms in this file.
