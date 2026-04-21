# ELF Security Sandbox & Model-Dedup Cache Protocol

**Companion to ELF Format Specification v0.1**
**Status:** Design draft
**Date:** 2026-04-20

---

## Part I — Security Sandbox Design

### 1. Threat model

Three adversaries, each with a distinct attack surface:

**A1. Malicious ELF publisher.** Distributes a crafted `.elf` that, when opened, attempts to:
- exfiltrate the host page's DOM, cookies, or local storage;
- exfiltrate the user's browsing data via network requests;
- escape the sandbox to run arbitrary code on the user's machine;
- persist state cross-session to track users;
- abuse the viewer's cache or fingerprinting surface.

**A2. Malicious host page.** Embeds a legitimate ELF and tries to:
- read the ELF iframe's DOM or chat transcripts;
- inject prompts or falsified responses visible to the user;
- spoof the ELF's identity (fingerprint, signature display);
- tamper with the ELF's storage or cache entries.

**A3. Network adversary.** On thin-ELF downloads (where the ELF references external model URLs):
- substitutes a malicious model at the declared URL;
- downgrades the connection;
- observes request patterns.

Non-goals (explicitly): preventing DRM circumvention, preventing a user who **wants** to extract the ELF's contents from doing so, preventing side-channel timing attacks that require co-resident execution.

### 2. Defense architecture

Four independent controls, any one of which is sufficient to prevent the most serious classes of attack. This is layered defense; removing any one layer should still leave a working security posture against the main threats.

```
Layer 1: Cross-origin iframe             →  stops A2 DOM access
Layer 2: sandbox="allow-scripts" (no     →  stops A1 from reaching host origin
         allow-same-origin against host)    storage / cookies
Layer 3: Strict CSP (connect-src 'none') →  stops A1 network exfiltration
Layer 4: SRI-style hash on every         →  stops A3 substitution
         resource (spec §7.1)
```

### 3. Layer-by-layer specification

#### 3.1 Layer 1 — Cross-origin iframe

The viewer MUST run at an origin distinct from the host page's origin. The Reference Viewer at `https://viewer.elf-format.org/` satisfies this automatically when embedded by any other site. Self-hosted viewers MUST be served from an origin dedicated to viewer duty, not from the host site's primary origin, to preserve cross-origin isolation from host scripts.

Consequence: browser same-origin policy prevents the host's scripts from reading the viewer iframe's DOM, and vice-versa, regardless of what either side tries.

#### 3.2 Layer 2 — Iframe sandbox

Embed iframe attributes:

```html
<iframe
  src="https://viewer.elf-format.org/embed?src=..."
  sandbox="allow-scripts"
  credentialless
  allow=""
  referrerpolicy="no-referrer"
  loading="lazy"
></iframe>
```

Notes:

- **`sandbox="allow-scripts"`** — scripts run, but the iframe is treated as a unique opaque origin. No cookies, no storage shared with the real viewer origin.
- **No `allow-same-origin`** — explicitly. The iframe cannot read cookies from the host OR from the viewer origin. Storage (OPFS, IndexedDB) inside the iframe is partitioned to the opaque origin and dies with the iframe.
- **`credentialless`** — opts the iframe into "anonymous" loading: no cookies sent with subresource requests, no cache reuse from the viewer origin's cached credentials.
- **`allow=""`** — no Permission Policy features granted (no camera, mic, geolocation, USB, etc.).
- **`referrerpolicy="no-referrer"`** — the ELF cannot learn the host URL via Referer.

Consequence of the opaque-origin choice: the ELF's OPFS storage is ephemeral. The **cache** lives in the viewer's real origin, not in the iframe — the viewer fetches and verifies resources, then hands byte blobs to the iframe via postMessage (or via srcdoc injection for small resources). The iframe never persists anything of its own; it's a pure compute surface.

#### 3.3 Layer 3 — Content Security Policy

Two CSPs apply: one for the outer viewer page, one for the inner ELF iframe.

**Outer viewer page CSP:**

```
default-src 'self';
script-src  'self' 'wasm-unsafe-eval';
style-src   'self' 'unsafe-inline';
img-src     'self' data: blob:;
font-src    'self';
connect-src 'self' https:;                 (must fetch ELFs from arbitrary origins)
worker-src  'self' blob:;
frame-src   blob:;                         (for the srcdoc-created iframe)
frame-ancestors *;                         (must be embeddable by any host)
form-action 'none';
base-uri    'self';
object-src  'none';
```

Plus:

```
Cross-Origin-Opener-Policy:   same-origin
Cross-Origin-Embedder-Policy: require-corp
```

These two are required for `SharedArrayBuffer` (needed by multi-threaded Wasm). `frame-ancestors *` is intentional — the viewer's purpose is to be embedded.

**Inner ELF iframe CSP** (applied via `<meta http-equiv>` inside the srcdoc, because srcdoc iframes can't carry their own HTTP response headers):

```
default-src 'none';
script-src  'self' blob: 'wasm-unsafe-eval';
worker-src  'self' blob:;
style-src   'self' 'unsafe-inline';
img-src     'self' data: blob:;
font-src    'self' data:;
media-src   'self' blob:;
connect-src 'none';
form-action 'none';
base-uri    'none';
frame-ancestors <viewer-origin>;
object-src  'none';
```

For thin ELFs, `connect-src 'none'` is replaced with the explicit list of declared `fetch_urls` origins:

```
connect-src https://huggingface.co https://cdn.example.com;
```

Why not just trust `default-src 'self'`? Because `'self'` in a srcdoc iframe is the opaque origin, and its meaning for `connect-src` is ambiguous across browsers. Explicit `'none'` is unambiguous.

#### 3.4 Layer 4 — Integrity verification

Every resource MUST be hash-verified before use (spec §7.1). For thin ELFs, the `fetch_urls` response bytes MUST be verified against the manifest-declared `sha256` before being handed to the runtime. A failing verification MUST be a hard abort with a user-visible error.

This is the defense against A3. It's also redundant protection against A1: a publisher who lies about a hash is caught on first verification, not at some later undefined moment.

### 4. Host ↔ ELF communication protocol

All communication between the host page and the ELF iframe goes through `window.postMessage`. The message surface is small and strictly schema-validated.

#### 4.1 Registered message types

```typescript
// Host → ELF
type HostToElf =
  | { type: 'elf/init';   manifestHash: string; hostOrigin: string }
  | { type: 'elf/prompt'; id: string; text: string; opts?: ChatOptions }
  | { type: 'elf/abort';  id: string }
  | { type: 'elf/terminate' }
  | { type: `x-${string}`; [k: string]: unknown };

// ELF → Host
type ElfToHost =
  | { type: 'elf/ready';     manifest: PublicManifest }
  | { type: 'elf/chat-delta'; id: string; delta: string; done: boolean }
  | { type: 'elf/error';     phase: string; message: string; recoverable: boolean }
  | { type: 'elf/resize';    width: number; height: number }
  | { type: `x-${string}`; [k: string]: unknown };
```

`PublicManifest` is a redacted subset of the manifest — title, description, authors, licenses, capabilities, fingerprint — without provenance details the ELF publisher may not want to expose to embedding sites.

#### 4.2 Validation rules (normative)

Every `postMessage` MUST be validated by the receiver before any action:

- `event.origin` MUST match the expected counterparty origin.
- `event.data` MUST be a plain object.
- `event.data.type` MUST be a known string or an `x-` prefixed extension.
- Unknown types MUST be logged and ignored, never forwarded.
- String lengths MUST be bounded (e.g., `text` ≤ 64 KiB in `elf/prompt`; `delta` ≤ 16 KiB in `elf/chat-delta`).
- Message rate MUST be bounded. The Reference Viewer enforces 100 msg/s per direction with a burst of 500.

#### 4.3 Explicit non-APIs

The host MUST NOT attempt to:

- Access `iframe.contentWindow.document` (blocked by cross-origin policy anyway).
- Send `eval`-style messages (no `elf/exec` type exists; extensions MUST NOT add one).
- Read the ELF's internal state beyond what `elf/chat-delta` reveals.

The ELF MUST NOT attempt to:

- Call `window.parent.*` beyond `postMessage`. Sandbox prevents most of this; CSP and `frame-ancestors` tighten it.
- Issue `window.top` navigations. Sandbox prevents this.
- Open new windows. `sandbox` lacks `allow-popups`.
- Submit forms. `sandbox` lacks `allow-forms`; `form-action 'none'` in CSP backstops.

### 5. Safety signals the viewer MUST surface

Required UI (spec §8.7 made concrete):

```
┌─────────────────────────────────────────────────────────────┐
│  📄 Example Knowledge Base                             [×]  │
│  by Sweet · 142 MB · opened from example.com                │
│  ────────────────────────────────────────────────────────    │
│  Fingerprint: 2f8a19…c4d3                                    │
│  Signature:   ✓ Verified  did:web:example.com                │
│  Network:     ❌ Offline (no connections permitted)           │
│  Capabilities granted: WASM, SIMD, WebGPU                    │
│  [ Details ] [ Clear state ] [ Terminate ]                   │
└─────────────────────────────────────────────────────────────┘
                  iframe containing ELF UI
```

Required elements, every time:

- **Title** from the manifest.
- **Fingerprint** — truncated for display, copyable in full.
- **Signature status** — "Verified ✓ `<identity>`" / "Invalid ✗" / "Unsigned".
- **Network status** — "Offline" for fully-local ELFs; full list of domains for thin ELFs.
- **Granted capabilities** — the actual capabilities the viewer provided, not merely what the ELF requested.
- **Terminate button** — kills the iframe and clears its state.

The viewer MUST NOT hide this chrome by default. It MAY be minimized by the user but MUST remain one click away.

### 6. Security incident response

When the viewer detects an incident (hash mismatch, signature failure, unexpected network attempt via CSP violation report), it MUST:

1. Halt the ELF immediately.
2. Display a red error banner with the specific cause.
3. Preserve the fingerprint and a copy of the manifest for reporting.
4. NOT automatically retry.
5. Offer a "Report this ELF" button that composes a shareable summary (fingerprint, source URL, user's viewer version) without any user-identifiable data beyond that.

---

## Part II — Model-Dedup Cache Protocol

### 7. Problem statement

If the average `.elf` embeds a 500 MB–2 GB model, and users open ten different ELFs that each happen to use TinyLlama-1.1B, downloading ten copies is untenable. The format is dead on adoption.

The solution: content-address every model weight by manifest-declared SHA-256, and have viewers share a cache keyed by that hash. Two ELFs that declare the same `language_model.sha256` share a single cached copy.

### 8. Design principles

1. **Content-addressed.** The cache key is the manifest-declared `sha256`, verified on first store and on every read.
2. **Viewer-scoped.** A browser's same-origin storage partitioning means the cache is per viewer-origin. This is a feature, not a bug: it prevents cross-origin cache-timing fingerprinting.
3. **Transparent to the ELF.** The ELF never knows whether a cache hit occurred. The runtime binding just receives a Blob.
4. **Bypassable.** A user MUST be able to inspect, evict, and disable the cache.
5. **Lossless under eviction.** If the cache evicts a resource that the ELF still needs, re-extraction from the ELF bytes MUST succeed.

### 9. Storage layout

OPFS is the primary backend. The viewer owns its origin's OPFS root and uses this layout:

```
/
├── elf-cache/
│   ├── v1/                        # schema version
│   │   ├── resources/
│   │   │   ├── 2f/
│   │   │   │   └── 8a19...c4d3    # file named by full sha256
│   │   │   ├── ...
│   │   │   └── index.json         # { sha256 → { size, firstSeen, lastUsed, refs } }
│   │   ├── elfs/                  # metadata for opened ELFs
│   │   │   └── <fingerprint>.json
│   │   └── config.json            # { maxSize, enabled, lastCleanup }
└── tmp/
    └── staging/                   # in-progress writes; atomically renamed on verify
```

Two-character directory sharding (`2f/8a19...`) keeps any single OPFS directory under a few thousand entries.

`index.json` is the authoritative map; individual resource files are truth-checked against it on access.

### 10. API

The `ModelCache` API exposed to the loader:

```typescript
interface ModelCache {
  /** Does a verified blob exist for this hash? */
  has(sha256: string): Promise<boolean>;

  /** Return a Blob for the hash, or null if not cached or corrupted. */
  get(sha256: string): Promise<Blob | null>;

  /**
   * Store a blob under its hash.
   * Atomically: writes to staging/, re-hashes, renames into resources/ only if verified.
   * Returns the final Blob (backed by the on-disk File) on success, throws on mismatch.
   */
  put(sha256: string, source: Blob | ReadableStream<Uint8Array>): Promise<Blob>;

  /** Mark that a specific ELF fingerprint references a hash. */
  addRef(sha256: string, elfFingerprint: string): Promise<void>;

  /** Remove a reference; eligible for eviction if refs = 0. */
  removeRef(sha256: string, elfFingerprint: string): Promise<void>;

  /** Manual eviction. */
  evict(sha256: string): Promise<void>;

  /** List everything cached, for the user-facing cache manager UI. */
  list(): Promise<CacheEntry[]>;

  /** Aggregate size on disk. */
  size(): Promise<number>;

  /** Configure cache cap. */
  setMaxSize(bytes: number): Promise<void>;
}

interface CacheEntry {
  sha256: string;
  size: number;
  firstSeen: string;     // ISO
  lastUsed: string;      // ISO
  refs: string[];        // ELF fingerprints that reference this
  displayName?: string;  // optional hint from manifests, e.g., "TinyLlama-1.1B Q4_K_M"
}
```

### 11. Cache-aware load flow

```
Loader.loadFromUrl(elfUrl)
    │
    ▼
fetch bytes, parse manifest, validate
    │
    ▼
For each model declared in manifest:
    │
    ├─ modelHash = manifest.language_model.sha256
    │
    ├─ if cache.has(modelHash):
    │     blob = await cache.get(modelHash)
    │     cache.addRef(modelHash, elfFingerprint)
    │     yield blob to runtime   ← INSTANT
    │
    └─ else:
          extract bytes from ELF (ZIP central dir or elf-resource script)
          stream them through:
            (a) SubtleCrypto.digest  (integrity check against manifest.sha256)
            (b) cache.put(modelHash, blob)   (OPFS staging → verify → rename)
          cache.addRef(modelHash, elfFingerprint)
          yield blob to runtime
```

### 12. Verification invariants

The cache enforces these invariants, asserted in tests:

1. **`put` is atomic.** A power-cut mid-write leaves staged bytes in `tmp/staging/`; on next boot, staging is purged. `index.json` is updated only after the rename into `resources/<hash>` succeeds.
2. **`get` re-verifies.** Every `get` re-hashes the file before returning a Blob if `config.verify_on_read` is true (default on first read since boot, off for subsequent reads in the same session). This catches OPFS corruption.
3. **Hash is the key is the truth.** A file named `resources/2f/8a19...c4d3` whose contents hash to anything other than `2f8a19...c4d3` is immediately evicted.
4. **Eviction is lazy-safe.** An ELF that's actively loading a model holds a read lock via a session-scoped in-memory ref counter. The disk-level `refs` array plus in-memory locks ensure a model isn't evicted from under a running session.

### 13. Eviction policy

Default: LRU with a 10 GB cap, user-adjustable.

```
On cache.put(sha256, blob):
    if cache.size() + blob.size > maxSize:
        candidates = cache.list()
            .filter(e => e.refs.length === 0)     # no ELF currently references
            .sort_by(e => e.lastUsed)             # oldest first
        while cache.size() + blob.size > maxSize and candidates.non_empty:
            evict(candidates.pop_front())
        if still over cap:
            throw CacheFullError                  # rare; user must intervene
```

### 14. Cross-ELF deduplication: the payoff

Consider:

```
User visits site A → opens handbook-A.elf (uses TinyLlama Q4_K_M, hash H1)
    → cold load: 630 MB download, write to cache
    → time-to-ready: ~60s on a 100 Mbps link, mostly fetch time

User visits site B → opens handbook-B.elf (also uses TinyLlama Q4_K_M, hash H1)
    → cache.has(H1) = true
    → skip extraction entirely
    → time-to-ready: ~2s (just corpus + runtime init)
```

This works even across different sites, because the cache lives at the viewer's origin, not at either host's origin. A single visit to *any* site embedding an ELF populates the cache; all subsequent visits anywhere using the same model hit warm.

### 15. Privacy considerations

Content-addressed caches are a known fingerprinting vector: an attacker can probe cache timing to learn which models a user has cached, which correlates with which ELFs they've opened. Mitigations:

- **Cache is per viewer origin.** An embedding site can't directly timing-probe the cache because its scripts don't run in the viewer origin.
- **No cross-origin API.** The cache is not exposed to the inner ELF iframe (which runs in an opaque origin); the viewer's loader consults the cache and hands Blobs over via postMessage/srcdoc.
- **Timing jitter on misses.** To thwart a cooperating host+ELF attempt at cache-membership inference via load-time oracle, the viewer MAY add randomized delays (e.g., 0–200 ms) on every `cache.get` regardless of hit status. Off by default; opt-in for privacy-conscious deployments.
- **User-facing controls.** The cache manager UI lists every cached resource with the ELF fingerprints that referenced it. Users can see and clear.

### 16. Thin-ELF fallback interaction

When an ELF declares `fetch_urls` for a model and the bytes are not embedded:

```
cache.has(sha256)?
    hit  → use cache (no network)
    miss → for url in fetch_urls:
              blob = fetch(url) with SRI-style post-verify
              if hash(blob) == sha256:
                  cache.put(sha256, blob)
                  break
           else:
              error: "No fetch URL yielded matching content"
```

This means a thin ELF still benefits from cache dedup exactly like a fat one — once any user has ever cached the model (from any ELF, thin or fat), subsequent thin-ELF opens are instant.

### 17. Cache configuration schema

```json
{
  "version": 1,
  "enabled": true,
  "max_size_bytes": 10737418240,
  "verify_on_read_first_time": true,
  "verify_on_read_always": false,
  "timing_jitter_ms": 0,
  "evict_on_quota_pressure": true,
  "user_allow_list": null,
  "user_block_list": []
}
```

`user_allow_list` and `user_block_list` let a user pin certain hashes (never evict) or forbid certain hashes (refuse to cache; refuse to use). Both are opt-in; default is unrestricted.

### 18. Telemetry: none

The viewer MUST NOT send cache telemetry (hit rates, hash lists, eviction counts) to any server. Any telemetry the viewer collects is local-only and viewable by the user in settings.

---

## Part III — Cross-cutting: what the whole thing buys you

With Parts I and II in place, the security properties an ELF publisher, host site, and user can each assume:

**Publisher can rely on:**
- Their corpus is delivered intact or not at all (hash integrity).
- Their signature, when present, is verified and displayed.
- Their ELF runs the same way on any conforming viewer.
- Their model is not re-downloaded for users who've seen it before — deployment is cheaper for popular models.

**Host site can rely on:**
- An ELF cannot steal their cookies, localStorage, or DOM.
- An ELF cannot initiate unapproved network traffic from their page.
- An ELF cannot spoof their content elsewhere.

**User can rely on:**
- An ELF runs with no network access unless they see a network indicator saying otherwise.
- Their chat with an ELF is partitioned — no cookies, no cross-ELF leakage.
- They can terminate any ELF and clear its state with one click.
- They can audit their model cache and clear it.
- They see a clear identity (signed or not, fingerprint) before trusting anything an ELF tells them.

**What's NOT guaranteed (honest disclosure):**
- An ELF *can* display anything visually inside its chrome. The viewer's chrome (title, fingerprint) is the only trustworthy UI; the ELF's own UI can claim anything.
- Signature verifies the *builder*, not the *truth* of the ELF's contents. A signed ELF may still contain wrong or malicious information.
- Timing and covert side channels are not addressed in 0.1.
- An adversarial model can produce adversarial outputs; nothing here prevents prompt-injection attacks mediated by the ELF's corpus. Defense-in-depth on that is application-layer.
