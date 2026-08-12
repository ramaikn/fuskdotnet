# Changelog

All notable changes to this project will be documented in this file.

## [1.2.0] - 2026-08-12
### Added
- **Safe / Max mode selector** (GUI dropdown + `--max` CLI flag). The mode governs only *how much a pass skips* — it never changes which passes run, so all feature checkboxes stay independent.
  - **Safe** (default): a pass leaves a site untouched whenever it cannot prove the transform is behaviour-preserving (1:1).
  - **Max**: runs every enabled pass at full coverage — the probabilistic hold-back is turned off so no eligible site is skipped. Non-optional skips (EH regions for the flattener; generic/vararg/value-type/inaccessible targets for the call proxy) are always kept, as forcing them emits invalid metadata or crashes at runtime.
- **Demand-scaled decryptor variants** — string, constant, and float decryptors now scale with the number of literals in the assembly (~1 per 2–5 literals, RNG-drawn) instead of a fixed 2–4, so literal-heavy assemblies get dozens of distinct, non-repeating decryptors.
- **Cleaner run log** — one plain-language line per pass summarising what was protected; no internal obfuscation parameters are exposed.

### Changed
- Self-obfuscated release binaries rebuilt with the expanded protection pipeline.

### Verified
- 50/50 obfuscation-integrity suite passes across reseeded builds in both Safe and Max modes; self-obfuscated obfuscator launches cleanly in both modes.

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
