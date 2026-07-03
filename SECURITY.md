# Security Policy

## Scope

This policy covers:

- flaws in the **specification itself** (identity computation, signature
  scheme, permission model, canonicalization, packaging profiles) that would
  allow a conforming artifact or executor to violate the security invariants
  in the Security and Cache Model document;
- vulnerabilities in the **reference viewer** and other code in this
  repository.

It does not cover vulnerabilities in third-party implementations; report
those to their maintainers. It does not cover malicious *content* of
individual ELF artifacts.

## Reporting

Report privately via **GitHub Security Advisories** on this repository
("Report a vulnerability"). Do not open a public issue.

Include, where possible: the affected document/section or file, a
description of the invariant violated, and a proof-of-concept artifact or
manifest if applicable.

## What to expect

- Acknowledgment within 7 days.
- An assessment of whether the issue is a specification flaw (requiring an
  erratum or version change) or an implementation bug.
- Coordinated disclosure: we ask for up to 90 days before public disclosure
  of specification-level flaws, since fixes may require changes across
  independent implementations.

## Specification-level advisories

Confirmed specification flaws are published as security errata listed in the
affected specification version and announced in the repository. Executors
implementing the affected version should treat published errata as
normative.

## Hall of thanks

Reporters of valid issues are credited (with permission) in the advisory.
No bounty program is currently offered.
