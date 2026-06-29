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

## Completed Actions
- Replaced silent `pass` in `get_source_hash` with `step_fail()` in `sagemake-template`.
- Replaced early exit in `check_dependencies` with complete error reporting in `sagemake-template`.
- Fixed unhandled exceptions in `cmd_install` and `cmd_clean` in `sagemake-template`.
- Added OS check before `.chmod()` in `sagemake`.
- Fixed Template Injection vulnerability in `sagemake`.
- Fixed Cache Collisions and Non-Deterministic Sorting in `sagemake-template`.
- Fixed Incremental Build Inaccuracy in `sagemake-template`.
