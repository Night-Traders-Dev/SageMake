# ForgeMaster Journal

## Architecture Analysis
- **Core Observation**: SageMake is not a traditional build system engine. It is a lightweight CLI framework and template generator that standardizes how build scripts are written and displayed.
- **Components**:
  - *Dependency graph engine*: None. Delegated to `make`/`cmake`.
  - *Scheduler*: None. Delegated to `make -j`.
  - *Cache system*: Minimal caching added natively via SHA256 of directory state.
  - *Artifact manager*: Minimal manual file copying.
- **Mechanism**: The `sagemake` script generates a `sagemake` file from `sagemake-template`. The generated script uses Python's `subprocess.run()` to sequentially execute shell commands.

## Major Discoveries
- **Correctness/Security**:
  - The original template correctly used `subprocess.run(check=True)` securely, avoiding silent failures.
  - *Template Injection Vulnerability*: User inputs (like project name and binary name) were injected directly into the `sagemake` Python script without escaping. This allowed arbitrary Python code execution if a payload contained double quotes and backslashes. **Fixed**: Introduced `escape_str` helper to safely escape strings before template substitution.
- **Determinism / Incremental Builds**:
  - *Cache Bug in Hashing*: A silent `pass` was used inside `try...except` during source hashing (`get_source_hash`). If reading a file failed, it would silently ignore it, leading to a determinism violation and incorrect cache hashes. **Fixed**: Replaced `pass` with `step_fail()` to ensure errors halt the process.
  - *Cache Collisions*: The hash function simply concatenated path bytes and content bytes. It was susceptible to collision attacks (e.g., path `a`, content `bc` vs path `ab`, content `c`). **Fixed**: Added null byte delimiters and 64-bit length prefixes for both path and content.
  - *Non-Deterministic Sorting*: `directory.rglob("*")` was sorted purely by `Path` objects, which compares case-insensitively on Windows but case-sensitively on Linux, leading to differing hashes. **Fixed**: Sorted using `key=lambda p: p.as_posix()`.
  - *Incremental Build Inaccuracy*: The system skipped builds solely if `.build_hash` matched. If the user deleted the output binary but left the hash file, the build falsely succeeded without creating a new binary. **Fixed**: Required `(BUILD_DIR / BINARY_NAME).exists()` to also be true before skipping the build.
- **Cross-Platform Issues**:
  - *Dependency Checking Bug*: `check_dependencies` exited early on the first missing dependency, which hurts Developer Experience. **Fixed**: Modified it to report all missing dependencies at once.
  - *Unhandled Exception in Install*: `cmd_install` created directories outside the `try...except` block, unhandling `PermissionError` when installing to restricted locations. **Fixed**: Moved directory creation inside the block.
  - *Unhandled Exception in Clean*: `cmd_clean` lacked exception handling during `shutil.rmtree()`, causing unhandled failures, especially on Windows when files might be locked. **Fixed**: Wrapped removal in `try...except`.
  - *Unsafe chmod*: `sagemake` called `.chmod()` without an OS check during generation. **Fixed**: Wrapped in `if os.name != "nt"`.

- *Path Traversal Vulnerabilities*: User inputs (like project name and binary name) were vulnerable to directory traversal attacks since `.` or `/` characters were allowed. **Fixed**: Added input validation blocking these characters in `sagemake`.
- *Performance/Scalability of Hashing*: `get_source_hash` loaded the entire file content into memory using `read_bytes()`, limiting scalability. **Fixed**: Modified hashing to process files in 8192-byte chunks using `st_size` for size prefixing.
- *Determinism around File Modes*: The source hash did not track file permission changes (like executable bit changes), leading to deterministic misses. **Fixed**: Included `st_mode` and `st_size` in the block to trace the source file's metadata.
- *Cache Corruption on Interruption*: Saving the source hash directly back into `.build_hash` could lead to a corrupt or half-written cache if interrupted. **Fixed**: Switched to using an atomic write strategy via `.tmp` file and `replace()`.
  - *Encoding UnicodeDecodeError on Windows*: Missing `encoding="utf-8"` on `read_text` and `write_text` would crash the generator on CP1252/Windows systems due to UTF-8 specific characters (ANSI/em-dashes). **Fixed**: Explicitly specified `encoding="utf-8"`.
  - *Hidden State Determinism Violation*: Changes in the build script (`sagemake`) itself did not invalidate the cache, leading to out-of-date builds if compiler flags or build logic changed. **Fixed**: Modified `get_source_hash` to natively hash the `sagemake` script itself alongside `src/`.
  - *Input Path Traversal Edge Cases*: Project and binary names weren't blocking `:` (Windows drive letters) or `.` (current directory), allowing edge case exploits/file overwrites. **Fixed**: Blocked these characters explicitly.
  - *Pathlib Null Byte Crashing Edge Cases*: Null bytes (`\0`) in inputs caused uncontrolled exceptions in standard Python filesystem functions. **Fixed**: Blocked null bytes explicitly.

## Completed Actions
- Added `encoding="utf-8"` to all `read_text` and `write_text` calls to support Windows.
- Modified `get_source_hash` to hash the script itself to prevent hidden state determinism bugs.
- Blocked `:` and `.` characters in project/binary names to fully patch path traversal vulnerabilities.
- Replaced silent `pass` in `get_source_hash` with `step_fail()` in `sagemake-template`.
- Replaced early exit in `check_dependencies` with complete error reporting in `sagemake-template`.
- Fixed unhandled exceptions in `cmd_install` and `cmd_clean` in `sagemake-template`.
- Added OS check before `.chmod()` in `sagemake`.
- Fixed Template Injection vulnerability in `sagemake`.
- Fixed Cache Collisions and Non-Deterministic Sorting in `sagemake-template`.
- Fixed Incremental Build Inaccuracy in `sagemake-template`.
- Added input validation for project name and binary name to prevent Path Traversal in `sagemake`.
- Modified `get_source_hash` to read files in chunks to improve scalability in `sagemake-template`.
- Added `st_mode` and `st_size` to the hash generation to detect permission changes in `sagemake-template`.
- Added atomic write via `.tmp` swap to `cmd_build` when writing to `.build_hash` to prevent Cache Corruption in `sagemake-template`.
