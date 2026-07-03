# Runtime & Distribution Licensing (GPL, launchers, and the "runs anywhere" vision)

This note covers three questions: whether GPL belongs anywhere in the ELF
stack, what the licenses of candidate launcher runtimes actually require,
and the legal/practical shape of a double-clickable universal launcher.

---

## 1. Should any ELF component be GPL?

Short version: **not the spec, not the schemas, maybe the flagship
launcher — and even there, probably not.**

### Where GPL would hurt

- **Spec prose / schemas / test vectors:** copyleft on these deters every
  commercial implementer. A format lives or dies on independent
  implementations. Keep CC BY 4.0 + Apache-2.0 as set up.
- **Reference viewer:** its whole purpose is to be copied from. Apache-2.0's
  patent grant is also doing security work here that GPLv2 doesn't.

### Where GPL is *coherent* (but still optional)

The **launcher application** is a standalone end-user product, like VLC.
GPL'ing it would:

- prevent proprietary forks of *your* launcher;
- **not** infect ELF artifacts it opens. Under the FSF's own
  interpretation, content interpreted or executed by a GPL program is not a
  derivative of that program (same reason GPL'd VLC can play any film and
  GCC can compile proprietary code). Artifacts are data to the launcher.

The costs: enterprise procurement friction, inability to ship on some
locked-down platforms cleanly, and one-way doors (you can relicense
Apache→GPL later; going GPL→Apache requires every contributor's consent).

### Recommended posture

Keep everything Apache-2.0. If you later want fork-protection for a
flagship runtime, **MPL-2.0** (file-level copyleft) gets most of the
benefit with far less adoption friction than GPL, and **AGPL** should be
avoided entirely unless you're deliberately running an open-core business
model. If contributors want to build GPL ELF tooling, they already can:
Apache-2.0 code is one-way compatible into GPLv3 projects, and CC BY 4.0
is officially declared one-way compatible with GPLv3 by Creative Commons.
Your current setup keeps the GPL door open for others without walking
through it yourself.

One trap to avoid: never let a hard **GPL (non-excepted) dependency** into
the launcher (classic example: GNU readline). That single dependency would
force the whole launcher to GPL.

---

## 2. License audit of candidate launcher stacks

The "double-clickable, runs anywhere" runtime, by stack:

| Component | License | Verdict |
|---|---|---|
| **V8** | BSD-3-Clause | Permissive. Bundle freely with attribution. |
| **Node.js** | MIT | Permissive. |
| **Chromium / Electron** | BSD / MIT | Permissive; large binary, include NOTICE aggregation. |
| **Tauri** | MIT / Apache-2.0 | Permissive; uses OS webview, small binaries. |
| **wasmtime / wasmer** | Apache-2.0 (wasmtime) | Permissive; cleanest Wasm path. |
| **llama.cpp / wllama** | MIT | Permissive. |
| **OpenJDK (Temurin)** | GPLv2 **with Classpath Exception** | Usable. The Classpath Exception is precisely what lets you bundle a jlink/jpackage runtime with a non-GPL app. Obligations: ship the JDK license texts, and be able to point to source for the OpenJDK bits (Temurin publishes it — a link suffices in practice). Do **not** redistribute Oracle's commercial JDK. |
| **GraalVM CE** | GPLv2 + Classpath Exception | Same as above. GraalVM EE/Oracle GraalVM has different terms — avoid redistributing. |

Conclusion: every serious path (Wasm-first via wasmtime, JS-first via
V8/Node/Tauri, JVM via Temurin) is compatible with a fully Apache-2.0
launcher. GPL is not forced on you by any of these; the JVM's GPL is
neutralized by the Classpath Exception as long as you use OpenJDK builds.

Ship a `THIRD-PARTY-NOTICES` file aggregating bundled licenses — Apache-2.0
§4(d) and the BSD/MIT attribution clauses all require it, and it's the
thing most hobby launchers forget.

---

## 3. The launcher architecture, legally

**Keep the artifact as inert data and the launcher as the signed
executable.** Resist the temptation to make the ELF file itself
double-clickably executable (self-extracting / polyglot binary):

- Mail gateways, browsers, and AV heuristics block or strip executables;
  a polyglot data-file/executable is indistinguishable from malware
  delivery technique and will be treated as such.
- Every executable artifact would need per-platform code signing; inert
  data files need none — ELF's own signature scheme (§4.14) covers content
  authenticity.
- Legally, an executable artifact makes each artifact *author* a software
  distributor (with the product-liability and CRA exposure that implies);
  an inert artifact keeps that on the launcher maintainer only, once.

This is the PDF/Office model: `.elf` registered as a file association, the
launcher handles double-click. The spec already supports this cleanly —
add a **native binding** (e.g. `"name": "native-launcher"`) alongside
`browser-js` in §4.8; nothing else in the core needs to change, which is
the payoff of the v0.2 layering.

### Platform distribution requirements (practical, not optional)

- **Windows:** Authenticode code-signing certificate (OV ~cheap, EV kills
  SmartScreen warnings faster). Unsigned launchers get the red warning.
- **macOS:** Apple Developer ID ($99/yr) + notarization, or Gatekeeper
  blocks it. File-association (UTI declaration) for `.elf` in Info.plist.
- **Linux:** no signing regime; AppImage/Flatpak both fine, both
  permissively licensed.

### Regulatory notes for shipping an executable

- **EU Cyber Resilience Act** (applies Dec 2027): a distributed launcher is
  a "product with digital elements." Non-commercial open source is largely
  carved out; the moment you monetize it (your startup instincts), you're
  in scope: vulnerability handling process, security updates, eventually CE
  marking. The SECURITY.md + SBOM-export hook already in this setup is the
  right skeleton.
- **US export controls:** publicly available open-source code is generally
  outside EAR scope (§734.7); standard TLS/crypto in a mass-market app is a
  non-issue. Only becomes real if you ship proprietary builds with custom
  crypto.
- **Liability:** the NOTICE disclaimer covers the repo; the launcher's
  installer/about screen should repeat the warranty disclaimer, because
  end users never see the repo.

---

## 4. "Any type of file" scope

Legally nothing changes — the resource table is content-agnostic already.
Two spec-side notes so the vision doesn't outgrow the text:

1. §0's framing ("language artifacts") will read as narrower than the
   ambition. When the scope genuinely broadens, rename the intro framing,
   not the invariants — identity, permissions, and hash-completeness are
   content-neutral and need no change.
2. Arbitrary file types raise the stakes on `media_type` honesty. Add a
   rule that executors MUST interpret resources per declared `media_type`
   and MUST NOT content-sniff — sniffing mismatched types is a classic
   security hole (and the browser profile should set
   `X-Content-Type-Options: nosniff` semantics for blob URLs).
