# Spec — `std.content` (content-addressing primitives: the identity model as a first-class library)

| Field | Value |
|---|---|
| **Status** | **Accepted** (2026-06-20, maintainer-ratified per DN-07 — guarantee matrix asserted in tests; open §7/§8 questions are design/scope calls, not contract violations; was *Implemented (Rust-first) — pending ratification* 2026-06-18, Draft/needs-design 2026-06-17) — the Rust-first code landed as `mycelium-std-content` (M-523, Batch P5-A; guarantee matrix asserted in tests). The Mycelium-lang migration (M-502-gated) remains. |
| **Module / Ring** | `std.content` (alias surface `hash`) · Ring 1 (RFC-0016 §4.2) · Tier A |
| **Tracks** | `M-523` (#164) — the Phase-5 task this spec delivers |
| **Scope** | The library form of Mycelium's **identity model**: read-only access to a value/definition's content-addressed identity (`hash-of-value`, `hash-of-AST`), digest equality, and the typed content-hash refs that cert/policy/provenance/`spore` artifacts already use. It re-exports and wraps the kernel's content-hash surface (RFC-0001 §4.6); it does not define a new hash. |
| **Boundary** | **NOT** hashing-for-hash-maps — keyed-bucket hashing (a `Hash`/`Hasher` trait for `Map`/`Set`) is a different, non-identity use of hashing and belongs to `collections` (M-511), not here. **NOT** the deployable/manifest layer — `spore` (M-522, ADR-013) *consumes* `content`'s digests but owns packaging. A representation change is `std.swap` (M-516); `content` never converts a value, only reports its identity. |
| **Depends on** | ADR-003 (content-addressed identity is canonical; metadata ≠ identity — `docs/Mycelium_Project_Foundation.md` §5 "Architecture Anchors", ADR-003); RFC-0001 §4.6 (the content-addressing rule + `hash(def)` formula) and §4.8 (formatting/serialization as projection); RFC-0016 §4.1 (the C1–C6 contract), §4.3 (`content` row), §4.5 (the guarantee matrix) |
| **Grounds on** | The Phase-1 content-addressing builder (**M-103**) and its `ContentHash` shape (`<algo>:<digest>`, `docs/spec/schemas/provenance.schema.json`); the `Provenance` schema (M-010, ratified). KC-3: this module is **above** the kernel — it adds no trusted hashing code. |

---

## 1. Summary

`std.content` is the identity model made into an ergonomic, read-only library. Its one load-bearing
promise is the **canonical-hash guarantee** (ADR-003 / RFC-0001 §4.6): a value's — or a definition's —
identity *is* the content-addressed digest of its **normalized** structure (and types-including-`Repr`,
and static contract), and **metadata is not identity**. The user-facing surface lets a program ask "what
is this thing's content hash?", "do these two things have the same identity?", and "give me a typed
content-ref to point a certificate / policy / `spore` at" — and nothing more. The structural honesty crux
is that the module **cannot** silently re-key identity: it exposes the kernel's canonical digest, so two
identically-normalized definitions **collide** (same hash — this is correct, not a bug), and a trivial
rename or reformat **does not** change identity (names/spans/formatting are not hashed). It is Ring 1, and
it adds **no trusted code** (KC-3): it re-exports / thinly wraps the Phase-1 content-hash surface (M-103);
all the trust lives in the kernel's normalizer + digest, which `content` only *reads*.

## 2. Scope & module boundary

- **In scope:** the typed `ContentHash` ref and its constructors/inspectors; `hash_of_value(&v)` (the
  content address of a runtime value, including its `Repr`); `hash_of_def(&def)` (the `hash(def)` of
  RFC-0001 §4.6 — the hash-of-AST identity); `digest_eq(a, b)` (identity equality by digest); building
  and resolving the content-refs that **cert / policy / provenance / spore** artifacts carry; the
  `hash ↔ name` lookup direction (resolve a name *to* an identity, read-only).
- **Out of scope (and who owns it):**
  - **Hashing-for-maps** — the `Hash`/`Hasher` trait used to bucket keys in `Map`/`Set`: that is a
    *non-identity* hash (it may be seeded, may be a non-cryptographic mixer, and is not content-addressed).
    Owned by **`collections` (M-511)**. Keeping this split explicit is a primary obligation of this spec
    (RFC-0016 §4.3 names `content` as "distinct from hashing-for-maps").
  - **Packaging / deployment** — the content-addressed *deployable* (a hash-identified DAG of code +
    config + manifest). Owned by **`spore` (M-522, ADR-013)**; it references `content`'s digests but owns
    the artifact.
  - **Representation change** — turning a value into another `Repr`. Owned by **`std.swap` (M-516)**;
    `content` is pure-observational and never mutates or converts.
  - **The hash algorithm itself** — the normalizer and digest are kernel/Phase-1 (M-103); `content`
    re-exports them, it does not re-implement them.
- **Ring & layering:** Ring 1 (Tier A capability surface). It **re-exports** the kernel's `ContentHash`
  type and **wraps** the kernel's `hash_of_*` entry points in an ergonomic, value-semantic surface. No new
  trusted code; no `wild`/FFI (ADR-014). KC-3 is preserved because the canonical digest stays in the
  kernel and this module is a pure *consumer* of it.

## 3. Exported-op surface (design sketch)

A read-only, value-semantic sketch — enough to fix the surface and feed the guarantee matrix, not a
committed grammar. `ContentHash` is the typed identity ref (the kernel's `<algo>:<digest>` shape, schema
`docs/spec/schemas/provenance.schema.json`); it is opaque, comparable, and serializable.

```
// illustrative signatures (not a committed surface)

// the typed identity ref — re-exported from the kernel, not minted here
opaque type ContentHash            // shape: "<algo>:<digest>" (M-103)

// --- identity of a runtime value (hash-of-value) ---
fn hash_of_value(v: &Value) -> ContentHash          // total: every value has an identity

// --- identity of a definition (hash-of-AST, RFC-0001 §4.6 hash(def)) ---
fn hash_of_def(d: &Def) -> ContentHash              // total: normalized structure ‖ types(+Repr) ‖ contract

// --- identity equality (NOT structural ==; digest equality) ---
fn digest_eq(a: &ContentHash, b: &ContentHash) -> Bool   // total, reflexive/symmetric/transitive

// --- typed refs for cert / policy / provenance / spore artifacts ---
fn as_ref(h: &ContentHash) -> ContentRef            // total: a typed pointer other modules embed
fn parse_ref(s: &Str) -> Result<ContentHash, MalformedDigest>  // C1: malformed shape is an explicit Err

// --- name <-> identity (read-only; names are metadata, not identity) ---
fn resolve_name(name: &Str) -> Option<ContentHash>  // None when the name is unbound (never a sentinel)
fn names_of(h: &ContentHash) -> List<Str>           // the hash -> name map direction (may be empty)
```

Notes on the sketch (each grounded, none committing detail the corpus has not fixed):
- `hash_of_value`/`hash_of_def`/`digest_eq` are **total** — every value/definition *has* an identity, and
  comparing two digests cannot fail. They are therefore `Exact` and not `Result`-returning. (C1 is met not
  by adding an error but because there is no failure mode to hide.)
- `parse_ref` is the one **fallible** op: a string that does not match the `<algo>:<digest>` shape is an
  explicit `Err`, never a silently-coerced or zeroed digest (C1).
- `resolve_name` returns `Option`, not a sentinel hash, because an unbound name is a legitimate "no
  identity here", not an error and not a fabricated digest (C1).

## 4. Guarantee matrix (the load-bearing deliverable — RFC-0016 §4.5)

Rows = exported ops. Encoded as a checked table (the RFC-0003 §4 template), asserted in tests once code
lands — never prose only.

| Op | Guarantee tag | Fallibility (explicit error set) | Declared effects | EXPLAIN-able? |
|---|---|---|---|---|
| `hash_of_value` | `Exact` (deterministic) | total | none | n/a |
| `hash_of_def` (hash-of-AST) | `Exact` (deterministic) | total | none | n/a |
| `digest_eq` | `Exact` (deterministic) | total | none | n/a |
| `as_ref` (cert/policy/spore ref) | `Exact` (deterministic) | total | none | n/a |
| `parse_ref` (cert-ref parse) | `Exact` (deterministic) | `Err(MalformedDigest)` on bad shape | none | n/a |
| `resolve_name` (name → identity) | `Exact` (deterministic) | `None` when name unbound | none | n/a |
| `names_of` (identity → names) | `Exact` (deterministic) | total (possibly empty list) | none | n/a |

Justification of the tags (all `Exact` — none weaker, so no `Proven`/`Empirical`/`Declared` to defend, VR-5):
- **Why every row is `Exact` / deterministic.** A content hash is a *pure function of normalized structure*
  (RFC-0001 §4.6, WF4: "its hash is a pure function of its normalized structure and types"); ADR-003 makes
  this the canonical, name-independent identity. There is no accuracy/precision/probability semantics
  anywhere here, so the lattice floor `Exact` applies directly (RFC-0016 C2: "an op with no accuracy
  semantics … is simply `Exact`"). Determinism is the substantive claim — identical normalized inputs
  always yield the same digest, and the digest never depends on names, spans, formatting, or dynamic
  metadata. This is the same determinism that makes provenance content-hashes form a stable DAG
  (RFC-0001 §4.6) and bounds content-addressable (RFC-0001 §4.7 "Determinism").
- **Why no effects.** Hashing reads its input and computes; it performs no IO, consults no clock, draws no
  randomness, and (the `hash ↔ name` map aside) touches no ambient mutable state. The name lookups
  (`resolve_name`/`names_of`) read a registry that is itself content-addressed/append-only; reads are
  effect-free observations. So every row declares `none` (C6 is met trivially — there is nothing to bound).
- **Why `EXPLAIN-able? = n/a`.** EXPLAIN/policy artifacts (C3) reify *selection / conversion / approximation*
  decisions. `content` does none of those — it neither selects a representation, converts, nor approximates;
  it reports a deterministic fact. A digest is its own witness (the identity *is* the hash), so there is no
  hidden decision to make inspectable. (This is the honest reading of C3, not a waiver — see §5/C3.)

## 5. §4.1 contract conformance (C1–C6)

- **C1 — never-silent (G2):** the three total ops cannot fail, so there is no failure to hide; the one
  shape-fallible op (`parse_ref`) returns `Err(MalformedDigest)` rather than a coerced/zeroed digest, and
  `resolve_name` returns `Option` (an unbound name is an honest `None`, never a sentinel hash). No silent
  clamp, no partial result.
- **C2 — honest per-op tag (VR-5):** every op is `Exact`/deterministic, the lattice floor, justified because
  none of them carry accuracy/precision/probability semantics (RFC-0016 C2). There is nothing to downgrade
  and nothing to overclaim — the matrix is the checked table, asserted in tests once code lands (§4.5).
- **C3 — no black boxes / EXPLAIN (SC-3/G11):** `content` *selects/converts/approximates* nothing, so the
  EXPLAIN obligation is `n/a` rather than unmet — there is no opaque decision behind a user-visible outcome.
  The dual human/machine projection (G11) still applies to the **ref**: a `ContentHash` renders to its
  canonical `<algo>:<digest>` string (machine) and resolves to its names (human) without either being its
  identity.
- **C4 — content-addressed, value-semantic (ADR-003):** this is the module's *raison d'être*. Identity is
  the content-addressed digest of the normalized AST/value (RFC-0001 §4.6); **metadata is not identity**
  (ADR-003 — names, spans, comments, formatting, provenance, measured sparsity, realized bounds,
  `policy_used` are all *not hashed*). Two identically-normalized definitions therefore **collide on the
  same hash** (correct), and a rename/reformat **preserves** identity (formatting is a projection,
  RFC-0001 §4.8). Two definitions differing only in `Repr` paradigm get **different** hashes (types include
  `Repr`, RFC-0001 §4.6). `ContentHash` is an immutable value; every op is a pure function of its inputs.
- **C5 — above the kernel (KC-3):** the normalizer + digest are the Phase-1 kernel surface (M-103); this
  module **re-exports the `ContentHash` type and wraps the `hash_of_*` entry points**, adding no trusted
  hashing code and no `wild`/FFI (ADR-014). The trusted base is unchanged.
- **C6 — declared, bounded effects (RFC-0014):** every op is effect-free (`none` in the matrix); there is
  no IO/time/randomness and no unbounded allocation to budget, so the declaration is "none" across the board.

## 6. Grounding

- **The canonical-hash guarantee + identity-vs-metadata split** — ADR-003 (Accepted; `docs/Mycelium_Project_Foundation.md`
  §5, ADR-003: "Unison-style content-addressing … formatting is a *projection*, not a mutation of identity")
  and RFC-0001 §4.6 (the `hash(def) = H(normalize(structure) ‖ types_with_repr ‖ static_contract)` rule;
  "Hashed (identity-bearing)" vs "Not hashed (metadata)"; "renaming does not change identity").
- **Determinism / collision semantics** — RFC-0001 §4.6 ("a pure function of its normalized structure") and
  WF4 (RFC-0001 §4.5); the `Repr`-sensitivity ("two definitions differing only in representation paradigm
  have different hashes") and the rename/reformat invariance ("a definition and its reformatting have the
  *same* hash").
- **The `ContentHash` shape + cert/policy/provenance/spore refs** — `docs/spec/schemas/provenance.schema.json`
  (`ContentHash` = `<algo>:<digest>`, M-103; `Provenance.Derived{op, inputs}` are content hashes;
  "Provenance is dynamic metadata and is NOT part of code identity"). `spore` consumes these refs
  (ADR-013 — "content-addressed definitions (ADR-003), shipped by hash").
- **The boundary vs `collections` hashing** — RFC-0016 §4.3 (`content` row: "distinct from hashing-for-maps"),
  §4.4 (`collections` is M-511).
- **Ring / contract / matrix obligations** — RFC-0016 §4.1 (C1–C6), §4.2 (Ring 1), §4.3 (`content` = M-523),
  §4.5 (the per-op guarantee matrix); KC-3 (above the kernel); VR-5 (honest tags); G2 (never-silent);
  G11 (dual projection).

## 7. Open questions (FLAGGED — resolve before ratification)

- **(Q1) The concrete digest API is not fixed by the corpus — described abstractly here.** The schema notes
  the `<algo>:<digest>` shape with `blake3` as an *example* and says "the concrete hash algorithm/encoding
  is fixed in Phase 1 (M-103); the pattern fixes only the shape" (`provenance.schema.json`). This spec
  therefore does **not** commit a specific digest algorithm, output width, or encoding for the exported
  surface — it wraps whatever M-103 fixes. **FLAGGED:** the exact `hash_of_value`/`hash_of_def` signatures
  (and whether `content` re-exports a single canonical algo or exposes an algo-tagged family) await the
  M-103 surface being pinned. Do not read "blake3" as committed.
- **(Q2) `hash_of_value` vs `hash_of_def` — the value/definition boundary.** RFC-0001 §4.6 specifies
  `hash(def)` (definition identity) in detail; the content address of an arbitrary *runtime value*
  (including its `Repr` and the r3 `Datum`/registry rules, RFC-0001 §4.6 r3) is implied by the value model
  but not spelled out as a single library entry point. **FLAGGED:** confirm with the maintainer that
  `hash_of_value` is a sanctioned, total exported op (vs. value identity being reachable only via
  serialization §4.8), and whether non-`Exact`/mixed-paradigm composite values have a defined identity or
  are an explicit refusal (cf. RFC-0001 §4.7 r3 "the interpreter refuses").
- **(Q3) The `hash ↔ name` map surface and its ownership.** RFC-0001 §4.6 stores names "separately as a
  `hash ↔ name` map". This spec exposes `resolve_name`/`names_of` read-only, but whether that registry lives
  behind `content`, behind `core`/the prelude, or behind the toolchain (LSP/registry) is not settled.
  **FLAGGED:** ties to RFC-0016 §8-Q2 (naming / module ownership). If the map is a toolchain artifact, these
  two ops move out of `std.content`.
- **(Q4) Ergonomics vs the contract for identity refs.** How implicit may `ContentHash` derivation be at a
  call site (e.g. an auto-derived identity for any value placed in a `spore`) vs. always-explicit?
  **FLAGGED:** the library instance of RFC-0016 §8-Q3 (ergonomics-vs-contract tension A) for this module.

## Meta — changelog

- **2026-06-17 — Draft (needs-design).** Stands up `std.content` (`hash`) — the **identity model as a
  first-class Ring-1 library** (M-523, #164): a read-only, value-semantic surface over the kernel's
  content-hash machinery (RFC-0001 §4.6; M-103) exposing `hash_of_value` / `hash_of_def` (hash-of-AST) /
  `digest_eq` / typed cert-policy-spore refs / read-only name resolution. The load-bearing artifact is the
  guarantee matrix (seven ops, **all `Exact`/deterministic, effect-free**), discharging C1 (never-silent —
  `parse_ref` → `Err`, `resolve_name` → `Option`, no sentinels), C2 (lattice-floor `Exact`, nothing to
  overclaim), C3 (`n/a` — `content` selects/converts/approximates nothing), C4 (the **canonical-hash
  guarantee** + the identity-vs-metadata split: identical normalized defs **collide**, renames/reformats
  **preserve** identity, `Repr` differences **change** it — ADR-003 / RFC-0001 §4.6/§4.8), C5 (above the
  kernel, re-exports/wraps M-103, no trusted code, KC-3) and C6 (no effects). Explicitly bounds the module
  **away from** hashing-for-maps (`collections`, M-511), packaging (`spore`, M-522), and representation
  change (`swap`, M-516). Four FLAGGED open questions (the not-yet-pinned concrete digest API — `blake3` is
  illustrative only; the value-vs-definition identity boundary; the `hash ↔ name` map ownership, tied to
  RFC-0016 §8-Q2; ergonomics-vs-contract, tied to §8-Q3) carried for ratification. No code; no kernel change
  (KC-3). Append-only.

- **2026-06-20 — Accepted (maintainer ratification, DN-07).** The maintainer ratified this Rust-first spec: the §4.5 guarantee matrix is asserted in tests, never-silent fallibility and honest per-op tags hold, and the open §7/§8 questions are design/scope calls, not contract violations. No guarantee tag was upgraded without a checked basis (VR-5). Status moves *Implemented (Rust-first) — pending ratification → Accepted*. Append-only; no kernel change (KC-3).
