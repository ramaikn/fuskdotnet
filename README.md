# Fusk .NET Obfuscator

A lightweight .NET obfuscator designed to resist static analysis while maintaining reliability and runtime performance, using dnlib.

![banner](banner.png)

## Table of Contents

- [Design Philosophy](#design-philosophy)
- [Key Features](#key-features)
- [Core Obfuscation Strategies](#core-obfuscation-strategies)
- [Usage Instructions](#usage-instructions)
- [Limitations and Scope](#limitations-and-scope)
- [License](#license)

## Design Philosophy

Fusk .NET Obfuscator operates under the principle of balancing reliability and runtime performance:

1. **Guaranteed Execution Semantics (1:1 Compatibility):** Every protection pass operates under a conservative safety model. If an IL construct, method structure, or reference cannot be definitively analyzed as safe for transformation, the pass skips it. Preserving application correctness is prioritized over maximum obfuscation coverage to eliminate runtime regressions.
2. **Zero Runtime Overhead:** Obfuscations target metadata and IL structures statically. The generated code runs at native speed without introducing heavy decryption loops, virtualization layers, or custom runtime interpreters that degrade execution performance.
3. **Modular Protection Pipeline:** Passes are executed sequentially in an isolated pipeline. Individual protections (such as String Encryption or Control Flow Flattening) can be toggled independently depending on your compatibility and security requirements.

## Key Features

Unlike naive obfuscators, Fusk .NET Obfuscator implements dynamic and polymorphic IL-level transformations:

- **Polymorphism Per Run:** All encryption keys, mathematical constant decoders, control flow state IDs, and block ordering permutations are derived from a random number generator initialized per build. Pushing the same source code twice yields distinct binary structures.
- **De-correlated Control Flow:** Basic blocks are segmented and linked via a switch dispatcher with randomized state numbering and physical memory layout, preventing straightforward disassembly flow recovery.
- **AV-Friendly Renaming:** Renames private types, methods, fields, and parameters using pronounceable pseudo-word combinations rather than high-entropy character sequences (like Unicode or base64 patterns) which trigger antivirus heuristics.
- **Zero-Friction Workflow:** Works on any CIL assembly (C#, VB.NET, F#, etc.) because it rewrites IL and metadata rather than source code. Supports direct file drag-and-drop or batch CLI execution to output the processed binary immediately beside the original.

## Core Obfuscation Strategies

| Strategy | Mechanism | Safety & Fallbacks |
|----------|-----------|--------------------|
| **Anti-Static Analysis** | Injects invalid metadata patterns and anti-decompilation IL structures to crash or break static decompilers (such as ILSpy, dnSpy) without affecting CLR loading. | Fully compatible with standard CLR and CoreCLR loaders; skipped on sensitive execution blocks. |
| **String Encryption** | Replaces static `ldstr` IL instructions with dynamic decryption helpers. The cipher key and decryption routine are generated randomly per build. | Covered by unit tests over empty, Unicode, and pathological strings. Behavior-preserving. |
| **Control Flow Flattening** | Reconstructs the method control flow graph into a flat state-machine driven by a `switch` dispatcher, randomizing state layout and execution paths. | Skipped on methods containing exception handlers, `switch`/`jmp` instructions, or active evaluation stacks at block boundaries. |
| **Symbol Renaming** | Renames internal and private classes, methods, fields, and parameters to low-entropy, pronounceable pseudo-words to avoid AV detection flags. | Retains public entry points, interop/P-Invoke interfaces, serialization models, and attributes. |
| **Constants Encryption** | Encrypts numeric, boolean, and floating-point constants into complex runtime mathematical expressions, hiding programmatic values. | Limited to safe numeric domains; does not alter compiler optimization blocks. |
| **Call Proxying** | Wraps direct method calls to external libraries or internal modules into dynamically resolved delegates, masking call-graph hierarchies. | Applied only to non-virtual, non-generic target calls that do not require reflection. |
| **Method Splitting** | Partitions method bodies into smaller, interconnected sub-procedures, distributing logical flow across generated internal helpers. | Excludes class constructors, async statemachines, and methods relying on strict stack structures. |
| **Metadata Shuffling** | Reorders physical metadata tokens and tables (types, methods, fields) within the PE binary, breaking tools that rely on sequential layout. | Relies on token-based CLR resolution; completely safe for any conforming .NET runtime. |

## Usage Instructions

1. **Download**: Get the latest compiled version from the [GitHub Releases](https://github.com/ramaikn/fuskdotnet/releases) page and extract the ZIP file.
2. **Graphical Interface**: Execute `FuskDotNet.exe`, drag and drop target assemblies onto the application window, configure the protections, and click **Obfuscate**.
3. **Command Line**: Run `FuskDotNet.exe path\to\App.exe [more.exe ...]` to obfuscate assemblies sequentially.

The processed output will be saved as `App_fuskdotnet.exe` in the same directory. The original file remains unmodified. For .NET Core/5+ applications, process the managed `App.dll` instead of the native host launcher.

## Limitations and Scope

- **Closed Source Freeware**: This is a closed-source freeware tool. The repository contains pre-compiled obfuscated releases under the `release/` folder.
- **Strong-Named Assemblies**: Output files are re-emitted without a valid signature since the original private key is unavailable. Sign the output binary after obfuscation if strong-naming is required by your target runtime environment.
- **Private Reflection**: Reflection-by-name over private members can fail due to renaming. Disable renaming if the target application relies on string-based reflection targeting private members.
- **Reversing Limits**: Control flow flattening increases the difficulty of static analysis but does not stop dynamic deobfuscation or runtime tracing.
- **Scope Limitations**: This tool is designed purely for code obfuscation. It does not perform packing, anti-debugging, anti-dumping, or code virtualization to maximize runtime stability and minimize false positives from antivirus software.

## License

This project is licensed under a proprietary Freeware License. See [LICENSE](LICENSE) for details.
