# Changelog

All notable changes to this project will be documented in this file.

## [1.5.0] - 2026-08-17

An execution-bound release. Where earlier versions hardened the binary against static folding and emulation, 1.5.0 goes further and forces automated static deobfuscation to become full dynamic execution of the whole application: the payload keys now only materialize when the program is genuinely run.

### Added
- **Runtime Image Key (RIK)** — every string and constant key mask folds in an extra always-zero term reconstructed at runtime through value-producing framework calls (`Math.Abs`, `BitConverter.DoubleToInt64Bits`). The term resolves correctly only when a real runtime executes the chain; the shortcuts a generic IL folder or emulator relies on — reading an unset field as its default, stubbing an unmodeled call to zero, or passing an argument straight through — each produce the wrong value and a corrupted key. It is independent of the existing module seed, so it also stands up to a symbolic solver, and costs a single field read per method. On by default.
- **Execution Path Key (EPK)** — the key mask of methods that run only after startup is tied to a value the program's entry point sets as its very first action. A tool that invokes a decryption routine in isolation — without running the program, the way `de4dot`'s dynamic string mode works — reads the default value and recovers the wrong key, so it must execute the entire application (for a GUI app, its full message loop) instead of a single function. A conservative reachability analysis limits the binding to methods provably reached only after startup, and the feature is automatically inert on library assemblies that have no entry point. On by default.
- **Poly-decoy injection** — decoy validation-shaped methods are injected and reached from a random subset of real methods behind a fixed-per-build guard that never runs. A reverse engineer hunting the real "decrypt-then-compare" logic now collides with many indistinguishable candidates a static tool cannot rule out. Binary growth is bounded by a configurable ratio (default 12% of eligible methods). On by default.

### Fixed
- **.NET Framework compatibility** — obfuscating a .NET Framework assembly no longer produces a binary that fails at startup with `TypeInitializationException` / `Could not load file or assembly 'System.Private.CoreLib'`. The injected runtime code (seed initializers, decryptors, and the exception-handler layer's `catch` type) referenced the framework types through the obfuscator's own runtime corlib; every such reference — including those reachable only through a method's exception-handler table — is now rewritten onto the target assembly's real corlib, so obfuscated output launches on every .NET Framework version as well as .NET 5+, with the full feature set (including the EH layer) enabled. Behaviour-neutral — the obfuscation strength is unchanged.

This revision completes the execution-bound plan: the primitives that earlier shipped only in their base form now carry their full layer set, and the anti-tamper and anti-localization pieces that were previously deferred are implemented and validated 1:1.

- **RIK self-identity + exception layers (A / C)** — the Runtime Image Key now also derives a term from the module's own version identity (read back at runtime and fixed at build time so it stays exactly zero on the untouched image) and can materialize part of its value inside a real exception handler. A folder or emulator that ignores reflection or exception dispatch computes the wrong value. The layer set is selectable (identity+value by default; the exception layer is opt-in because a throw in a static constructor is antivirus-heuristic sensitive).
- **Self-integrity** — an optional two-stage image hash: after the assembly is written, a stable region of the produced file is hashed and the value is baked back in, consumed everywhere as an always-zero term. A tool that rewrites or repacks the assembly shifts the bytes, the runtime hash stops matching, and every key is corrupted. On the untouched original the term is exactly zero (1:1). Opt-in; assumes a .NET target and falls back to a no-op where the image has no on-disk location.
- **MVID anti-tamper** — optionally binds the module version identity into the runtime seed relation as well, so a tool that regenerates the assembly's identity breaks the opaque predicates and the control-flow dispatcher, not only the string/constant keys. Opt-in.
- **Call-proxy target table** — a proxied call routes through a dispatcher whose leaf-target index is selected by an always-zero runtime term, so a static tool sees several candidate targets behind a mask it cannot fold and can no longer map one call site to one forwarder. On by default.
- **Multi-head dispatcher** — the control-flow dispatcher is emitted at several identical locations per method and each transition jumps to a random one, defeating the "the block with the most incoming branches is the dispatcher head" heuristic a de-flattener uses to locate it. Delegate/event handler targets are now always flattened regardless of the coverage setting, so a handler located via its wiring never reads as a linear statement list.
- **New GUI toggles** — *Runtime Image Key* (+ *EH Layer* sub-option), *Execution Path Key*, *Poly Decoys*, *Proxy Table*, *Self-Integrity*, and *MVID Guard* checkboxes expose every protection per run; the panel was reorganized into five columns with uniform spacing.

## [1.4.0] - 2026-08-16

A hardening release aimed at a cheaper, more realistic attacker than the previous one: the **concrete/interpretive evaluator** that runs the seed initializer through a simple IL interpreter instead of solving the seed relation algebraically. Where 1.3.0 raised the ceiling for a symbolic solver, 1.4.0 closes the gap for an evaluator that just executes the code.

### Added
- **Self-referential dispatcher state** — the control-flow dispatcher now folds an always-zero expression over its own *live, executed* recombined state into the switch input before decoding. Resolving the switch index therefore requires knowing which state actually ran, while recovering that state is itself hidden behind the seed-relation opaque term — a circular dependency that stops the linear "fold the predicate, then de-flatten" pipeline any evaluator relies on.
- **Broadened identity-call surface** — the string/constant decryptor lens now draws 1–3 operations at random per site from a much larger pool of universal, deterministic **identity** calls (`Convert.ToInt32/ToInt64`, `Math.Max/Min` against a value and against the type bounds, integer widen/narrow round-trips, and a bit-exact round-trip through the floating-point domain) instead of a fixed handful. A generic emulator that stubs unmodeled calls to a default now corrupts the value across many more shapes and must model far more of the runtime to keep up.
- **Diversified always-zero skeletons** — opaque expressions and per-method key masks now select from several structurally-independent zero identities (carry, union, xor-swap, and/or-sum, symmetric-difference, shift-sum) and fold in two independent skeletons per site, rather than varying only the coefficients of one template. A pattern matcher keyed on the *shape* of the identity no longer generalises from one recognised form.
- **Distributed seed derivation** — the runtime-seed relation's constants are split across several independent injected host types (a fragment parked in a static field on a separate host, reconstructed in the seed initializer) instead of living in one static constructor. Recovering the relation now means tracing multiple types and their initialization order — more so when hosts are disguised as ordinary user types.
- **Higher-degree and combined relations** — opaque expressions can now be raised to degree-3 (a zero residue multiplied by two seed loads), and per-method key masks combine two independent seed relations by XOR. Folding a masked site can require recovering more than one relation and reasoning over a seed product, not just a linear combination.

## [1.3.0] - 2026-08-16

A hardening release aimed at the ceiling against **automated static deobfuscation**: it removes the fixed structural signatures such tools rely on and forces the control-flow dispatcher to require multi-variable analysis.

### Added
- **Dual-coupled control-flow state** — the switch dispatcher's state is split across two coupled locals (`state = enc ^ r`, `state2 = r` for a fresh per-transition `r`), recombined only at the dispatcher. A deflattener that tracks a single state variable — the standard shape — can no longer recover the switch index; the reconstruction must discover the pairing across both locals.
- **Polymorphic injected-host shape** — every generated helper container (runtime seeds, string/constant decryptors, the string pool, forwarder-stub hosts) now emits with randomised class attributes instead of the fixed `static class` (`sealed + abstract`) shape. That fixed pair was a free structural fingerprint an automated deobfuscator keys on to locate obfuscator artifacts; because the runtime-seed host is the pivot the whole seed-based chain (opaque predicates, per-method key masks, control-flow dispatcher) hangs off, removing that certain match forces an attacker onto a lossy heuristic (false positives on ordinary user static classes) or the expensive interprocedural recovery of the degree-2 seed relation.
- **Randomised seed field width** — the runtime-seed fields use a per-field-random integer width, so they no longer match a fixed field-type signature.
- **Diversified call-proxy hosts** — forwarder stubs scatter across a small pool of differently-shaped hosts rather than one, so a proxy no longer maps to a single fingerprintable container.

### Added — Scenario A hardening (raises the static-deobfuscation ceiling)
- **Stealth host placement** — injected helper containers (runtime seeds, decryptors, string pool, forwarder stubs) no longer share the tell-tale shape *empty namespace + `Object` base + only-static-members + no instance constructor* that an automated deobfuscator keys on to locate obfuscator artifacts. Each host now reads like an ordinary user class: a real (scattered) namespace, a decoy instance constructor, and decoy instance fields. Renamed top-level user types are scattered across real namespaces instead of collapsing into the single empty-namespace node.
- **Decryptor key chaining** — each per-method key mask now folds in a call that returns 0 only under the runtime seed relation, so the mask's initialiser is no longer the pure arithmetic a static tool can constant-fold to recover the key. Folding a keyed string/constant site now requires emulating the call and the seed relation together, dissolving the per-site isolation that let a tool decrypt each literal independently.
- **Wide-BCL cipher lens** — the string/constant decryptors thread their running value through universal, deterministic **identity** BCL calls (`Convert.ToInt32/ToInt64`, `Math.Max/Min` of a value with itself) that a generic IL emulator does not model. An emulator that stubs unknown calls to a default silently produces garbage and must grow toward a full CLR to keep up.
- **Metadata-bound seeds** — the runtime seeds are mixed with `typeof(host).MetadataToken`, so a tool that rewrites the assembly shifts the tokens and an emulator that ignores `ldtoken`/reflection computes the wrong seeds — every opaque expression becomes non-zero and deflattening breaks.
- **Data-dependent dispatcher state** — the control-flow dispatcher's stored state now mixes in a live method parameter through an always-zero MBA term over the seed relation, so static state resolution needs data-flow over user code, not a constant fold.

## [1.2.0] - 2026-08-12
### Added
- **Safe / Max mode selector** (GUI dropdown + `--max` CLI flag). The mode governs only *how much a pass skips* — it never changes which passes run, so all feature checkboxes stay independent.
  - **Safe** (default): a pass leaves a site untouched whenever it cannot prove the transform is safe.
  - **Max**: runs every enabled pass at full coverage — the probabilistic hold-back is turned off so no eligible site is skipped. Non-optional skips (EH regions for the flattener; generic/vararg/value-type/inaccessible targets for the call proxy) are always kept, as forcing them emits invalid metadata or crashes at runtime.
- **Demand-scaled decryptor variants** — string, constant, and float decryptors now scale with the number of literals in the assembly (~1 per 2–5 literals, RNG-drawn) instead of a fixed 2–4, so literal-heavy assemblies get dozens of distinct, non-repeating decryptors.
- **Cleaner run log** — one plain-language line per pass summarising what was protected; no internal obfuscation parameters are exposed.

### Changed
- Self-obfuscated release binaries rebuilt with the expanded protection pipeline.

## [1.1.0] - 2026-08-12
### Added
- **Metadata Cleanup pass**: randomises the module MVID per build and blanks parameter
  names on rename-safe members (public/reflected surface keeps its names).
- **Live-value opaque predicates**: predicate guards now mix a live method argument into an
  always-zero MBA term, so a dynamic partial-evaluator can no longer fold them by reading the
  three static seed fields alone.
- **Opaque-predicate shape variation**: guards emit in both `brtrue` (bogus-at-end) and
  `brfalse` (always-true, bogus-inline) senses, so the guard is not a single templatable marker.
- **Bogus dispatcher arms**: control-flow flattening appends unreachable `switch` cases
  (states the codec never produces), breaking the "one arm per real block" topology a
  deflattener keys on.
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
