# Changelog

All notable changes to this project will be documented in this file.

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
