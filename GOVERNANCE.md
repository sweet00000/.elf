# Governance

This document defines who controls the specification and its registries.
Section 9.3 of the specification declares certain value spaces
registry-managed; this is that registry process.

## 1. Editors

The specification is maintained by its Editors. The current Editors are
listed in `EDITORS.md`. Editors merge changes, publish versions, and rule on
registry additions. New Editors are appointed by consensus of the current
Editors.

If the project is later transferred to a standards organization, that
organization's process supersedes this document, subject to the constraint
in `CONTRIBUTING.md` §1(4) that terms remain royalty-free for implementers.

## 2. Registries

The authoritative registries live in this repository under `/registries`,
one JSON file per registry:

- `roles.json` — resource roles
- `operations.json` — operation names
- `fulfillment-kinds.json` — fulfillment kinds
- `requirements.json` — requirement tokens
- `permissions.json` — permission tokens
- `bindings.json` — binding names
- `template-formats.json` — template format names

Each entry records: the token, the specification version in which it was
registered, a one-line definition, and a link to its normative definition.

## 3. Registration policy

- **Additions** are made by pull request and require Editor approval plus a
  written normative definition (either in the spec or in a linked stable
  document). Additions are backward-compatible by construction and take
  effect without a version bump.
- **Removals or semantic changes** to a registered value are
  backward-incompatible and require a major version change.
- **Provisional use.** Anyone may use `x-` prefixed or URN-namespaced values
  without registration. Registration is only required to claim an
  unprefixed token.
- **Denial.** Editors may decline entries that are duplicative, unsafe, or
  insufficiently specified; the reason is recorded in the PR.

## 4. Versioning authority

Only Editors may publish a specification version. A published version is
immutable except for errata (see `CONTRIBUTING.md` §4 and `SECURITY.md`).

## 5. Naming

The format's expanded name is **Embedded Language Format** in all documents.
Earlier drafts used variant expansions; those are historical. The name is
unrelated to the Executable and Linkable Format used for compiled binaries,
and documents should include a disambiguation note where confusion is
plausible.
