# Changelog

All notable changes to this project will be documented in this file.

## [1.4.0] - 2026-08-16

A hardening release aimed at a cheaper, more realistic attacker than the previous one: the **concrete/interpretive evaluator** that runs the seed initializer through a simple IL interpreter instead of solving the seed relation algebraically. Where 1.3.0 raised the ceiling for a symbolic solver, 1.4.0 closes the gap for an evaluator that just executes the code.

### Added
- **Self-referential dispatcher state** — the control-flow dispatcher now folds an always-zero expression over its own *live, executed* recombined state into the switch input before decoding. Resolving the switch index therefore requires knowing which state actually ran, while recovering that state is itself hidden behind the seed-relation opaque term — a circular dependency that stops the linear "fold the predicate, then de-flatten" pipeline any evaluator relies on. The term is zero for every value, so behaviour is unchanged (1:1).
- **Broadened identity-call surface** — the string/constant decryptor lens now draws 1–3 operations at random per site from a much larger pool of universal, deterministic **identity** calls (`Convert.ToInt32/ToInt64`, `Math.Max/Min` against a value and against the type bounds, integer widen/narrow round-trips, and a bit-exact round-trip through the floating-point domain) instead of a fixed handful. A generic emulator that stubs unmodeled calls to a default now corrupts the value across many more shapes and must model far more of the runtime to keep up. Every call is an exact identity, so the decrypted value is unchanged (1:1) and every method resolves in the target's own corlib on all runtimes.
- **Diversified always-zero skeletons** — opaque expressions and per-method key masks now select from several structurally-independent zero identities (carry, union, xor-swap, and/or-sum, symmetric-difference, shift-sum) and fold in two independent skeletons per site, rather than varying only the coefficients of one template. A pattern matcher keyed on the *shape* of the identity no longer generalises from one recognised form.
- **Distributed seed derivation** — the runtime-seed relation's constants are split across several independent injected host types (a fragment parked in a static field on a separate host, reconstructed in the seed initializer) instead of living in one static constructor. Recovering the relation now means tracing multiple types and their initialization order — more so when hosts are disguised as ordinary user types. The reconstructed values are bit-for-bit identical, so the relation and behaviour are unchanged.
- **Higher-degree and combined relations** — opaque expressions can now be raised to degree-3 (a zero residue multiplied by two seed loads), and per-method key masks combine two independent seed relations by XOR. Folding a masked site can require recovering more than one relation and reasoning over a seed product, not just a linear combination.

### Notes
- All changes are behaviour-identical (1:1) and verified against the project's internal correctness test suite (Safe and Max modes) plus a self-obfuscation run. Every added construct contributes exactly zero / is an exact identity, so the output computes the same result as the input.

## [1.3.0] - 2026-08-16

A hardening release aimed at the ceiling against **automated static deobfuscation**: it removes the fixed structural signatures such tools rely on and forces the control-flow dispatcher to require multi-variable analysis.

### Added
- **Dual-coupled control-flow state** — the switch dispatcher's state is split across two coupled locals (`state = enc ^ r`, `state2 = r` for a fresh per-transition `r`), recombined only at the dispatcher. A deflattener that tracks a single state variable — the standard shape — can no longer recover the switch index; the reconstruction must discover the pairing across both locals. Seed-independent, so it holds even if the seed fields are identified. Purely a reversible identity (`r` cancels), so behaviour is unchanged (1:1).
- **Polymorphic injected-host shape** — every generated helper container (runtime seeds, string/constant decryptors, the string pool, forwarder-stub hosts) now emits with randomised class attributes instead of the fixed `static class` (`sealed + abstract`) shape. That fixed pair was a free structural fingerprint an automated deobfuscator keys on to locate obfuscator artifacts; because the runtime-seed host is the pivot the whole seed-based chain (opaque predicates, per-method key masks, control-flow dispatcher) hangs off, removing that certain match forces an attacker onto a lossy heuristic (false positives on ordinary user static classes) or the expensive interprocedural recovery of the degree-2 seed relation.
- **Randomised seed field width** — the runtime-seed fields use a per-field-random integer width. They load and arithmetise identically on the IL stack (zero runtime cost, relation unchanged), but no longer match a fixed field-type signature.
- **Diversified call-proxy hosts** — forwarder stubs scatter across a small pool of differently-shaped hosts rather than one, so a proxy no longer maps to a single fingerprintable container.

### Added — Scenario A hardening (raises the static-deobfuscation ceiling)
- **Stealth host placement** — injected helper containers (runtime seeds, decryptors, string pool, forwarder stubs) no longer share the tell-tale shape *empty namespace + `Object` base + only-static-members + no instance constructor* that an automated deobfuscator keys on to locate obfuscator artifacts. Each host now reads like an ordinary user class: a real (scattered) namespace, a never-called decoy instance constructor, and decoy instance fields. Renamed top-level user types are scattered across real namespaces instead of collapsing into the single empty-namespace node (itself a "this is obfuscated" signal). Behaviour-neutral: the members the helpers use stay static, the decoy ctor is never invoked.
- **Decryptor key chaining** — each per-method key mask now folds in a call that returns 0 only under the runtime seed relation, so the mask's initialiser is no longer the pure arithmetic a static tool can constant-fold to recover the key. Folding a keyed string/constant site now requires emulating the call and the seed relation together, dissolving the per-site isolation that let a tool decrypt each literal independently. `key ^ 0 == key`, so 1:1.
- **Wide-BCL cipher lens** — the string/constant decryptors thread their running value through universal, deterministic **identity** BCL calls (`Convert.ToInt32/ToInt64`, `Math.Max/Min` of a value with itself) that a generic IL emulator does not model. An emulator that stubs unknown calls to a default silently produces garbage and must grow toward a full CLR to keep up. The calls are exact identities, so the value is unchanged (1:1) and resolve in the target's own corlib on every runtime.
- **Metadata-bound seeds** — the runtime seeds are mixed with `typeof(host).MetadataToken`. The degree-2 relation is recomputed from the metadata-derived seeds in the same static constructor, so it still holds exactly at runtime (1:1), but a tool that rewrites the assembly shifts the tokens and an emulator that ignores `ldtoken`/reflection computes the wrong seeds — every opaque expression becomes non-zero and deflattening breaks.
- **Data-dependent dispatcher state** — the control-flow dispatcher's stored state now mixes in a live method parameter through an always-zero MBA term over the seed relation, so static state resolution needs data-flow over user code, not a constant fold. The term is 0 for every parameter value, so behaviour is unchanged.

### Notes
- All changes are behaviour-identical (1:1) and verified against the project's internal correctness test suite (Safe and Max modes) plus a self-obfuscation run. Injected-host variants keep every member static, so the type is never instantiated (`Abstract` needs no ctor; `Sealed`-only mirrors the compiler's own `<PrivateImplementationDetails>`); stealth placement adds only a never-called decoy constructor.

## [1.2.0] - 2026-08-12
### Added
- **Safe / Max mode selector** (GUI dropdown + `--max` CLI flag). The mode governs only *how much a pass skips* — it never changes which passes run, so all feature checkboxes stay independent.
  - **Safe** (default): a pass leaves a site untouched whenever it cannot prove the transform is behaviour-preserving (1:1).
  - **Max**: runs every enabled pass at full coverage — the probabilistic hold-back is turned off so no eligible site is skipped. Non-optional skips (EH regions for the flattener; generic/vararg/value-type/inaccessible targets for the call proxy) are always kept, as forcing them emits invalid metadata or crashes at runtime.
- **Demand-scaled decryptor variants** — string, constant, and float decryptors now scale with the number of literals in the assembly (~1 per 2–5 literals, RNG-drawn) instead of a fixed 2–4, so literal-heavy assemblies get dozens of distinct, non-repeating decryptors.
- **Cleaner run log** — one plain-language line per pass summarising what was protected; no internal obfuscation parameters are exposed.

### Changed
- Self-obfuscated release binaries rebuilt with the expanded protection pipeline.

## [1.1.0] - 2026-08-12
### Added
- **Metadata Cleanup pass**: randomises the module MVID per build and blanks parameter
  names on rename-safe members (public/reflected surface keeps its names). Metadata-only,
  so it is behaviour-neutral (1:1).
- **Live-value opaque predicates**: predicate guards now mix a live method argument into an
  always-zero MBA term, so a dynamic partial-evaluator can no longer fold them by reading the
  three static seed fields alone.
- **Opaque-predicate shape variation**: guards emit in both `brtrue` (bogus-at-end) and
  `brfalse` (always-true, bogus-inline) senses, so the guard is not a single templatable marker.
- **Bogus dispatcher arms**: control-flow flattening appends unreachable `switch` cases
  (states the codec never produces), breaking the "one arm per real block" topology a
  deflattener keys on. Never selected at runtime (1:1).
- **Expanded call indirection**: proxying now also covers static/instance field access
  (`ldsfld`/`stsfld`/`ldfld`/`stfld`) and `newobj` construction for accessible, non-generic,
  same-module targets — plus an opaque fake-conditional inside forwarder stubs so they are no
  longer trivially inlinable. `initonly` writes and address loads (`ldflda`) are skipped.
- **String-pool coverage knob**: a configurable fraction of encrypted string sites decrypt
  inline instead of through the shared pool, so the pool is not a single one-pass dump target.
- **Headless `--quiet` batch mode**: obfuscate from the command line with no dialog; the exit
  code and a `fusk-report.txt` carry the result (useful for CI / scripted builds).

### Changed
- Self-obfuscated release binaries rebuilt with the expanded protection pipeline.

## [1.0.0] - 2026-08-12
### Added
- Initial public release of FuskDotNet.
- Core obfuscation engine powered by dnlib.
- Main protections:
  - **String Encryption**: Encrypts literial strings to resist static string searches.
  - **Control Flow Flattening**: Flattens control flows using switches to make disassembly hard to read.
  - **Symbol Renaming**: Renames symbols to pseudo-words to prevent decompilation mapping.
  - **Anti-Static Analysis**: Basic anti-static protections.
- User-friendly WinForms interface and CLI execution support.
- Configured layout to display **Anti-Static** checkbox at the top as the primary feature.
