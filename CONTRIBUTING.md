# Contributing to the ELF Format Specification

Thanks for your interest. This document covers the legal terms of
contributing and the process for changing the specification. Read it before
opening a pull request — PRs that don't meet the sign-off requirement below
cannot be merged.

## 1. Inbound licensing (what you agree to)

By submitting a contribution you agree that:

1. Your contribution to **specification prose** is licensed under
   **CC BY 4.0** (see `LICENSE-DOCS.md`).
2. Your contribution to **code, schemas, or test material** is licensed
   under the **Apache License 2.0** (see `LICENSE`), including its Section 5
   (submission of contributions).
3. The patent commitment in `PATENTS.md` applies to your contribution.
4. The editors may republish the specification, including your contribution,
   through a standards organization (for example a W3C Community Group or the
   IETF) under that organization's document license and IPR policy, provided
   the terms remain royalty-free and no more restrictive for implementers
   than the terms above.

Item 4 exists so the specification can graduate to a standards body later
without needing to re-contact every past contributor.

## 2. Developer Certificate of Origin (required)

Every commit must be signed off (`git commit -s`), which adds a
`Signed-off-by: Your Name <email>` trailer certifying the
Developer Certificate of Origin 1.1:

```
Developer Certificate of Origin
Version 1.1

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I
    have the right to submit it under the open source license
    indicated in the file; or

(b) The contribution is based upon previous work that, to the best
    of my knowledge, is covered under an appropriate open source
    license and I have the right under that license to submit that
    work with modifications, whether created in whole or in part
    by me, under the same open source license (unless I am
    permitted to submit under a different license), as indicated
    in the file; or

(c) The contribution was provided directly to me by some other
    person who certified (a), (b) or (c) and I have not modified
    it.

(d) I understand and agree that this project and the contribution
    are public and that a record of the contribution (including all
    personal information I submit with it, including my sign-off) is
    maintained indefinitely and may be redistributed consistent with
    this project or the open source license(s) involved.
```

Sign-offs must use a name you are known by; anonymous or clearly fictitious
sign-offs cannot be accepted for normative changes.

## 3. Specification change process

- **Editorial changes** (typos, clarifications with no normative effect):
  open a PR; an editor merges.
- **Normative changes** (anything altering a MUST/SHOULD/MAY, a registry,
  identity computation, or the security model): open an issue first
  describing the problem, the proposed change, and its compatibility impact.
  Normative changes land in the next draft version, never silently into a
  published version.
- **Registry additions** (new roles, operations, fulfillment kinds,
  requirement/permission tokens, binding names, template formats): see
  `GOVERNANCE.md`. Registry additions are additive and do not require a
  version bump.

## 4. Versioning of the spec documents

Published versions (`ELF-SPEC-vX.Y.md`) are immutable except for errata,
which must be listed in a change log within the document. Work happens in
the next version's directory.

## 5. Security issues

Do not report security-relevant spec flaws or reference-viewer
vulnerabilities in public issues. See `SECURITY.md`.
