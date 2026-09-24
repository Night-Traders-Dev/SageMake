# ForgeMaster Journal

## Major Discoveries

### Build graph edge cases
- `sagemake-template` generated projects use a single `build` command rather than full dependency graph parsing. However, when integrated with complex dependencies, `check_dependencies` runs linearly.
- SageOS and SageVM incorporate other repositories as Git submodules or dependencies and coordinate across them via subprocess.

### Scheduler limitations
- No built-in parallelism in the Python wrapper logic in the template (relies on underlying `make -j` or similar).

### Cache bugs
- The template uses SHA-256 for caching in `get_source_hash`, which includes the script itself, OS/arch info, env vars, file paths, executable bits, file sizes, and file contents. It uses atomic writes to prevent race conditions during updates.
- If a user runs `build` without any arguments, `args` defaults to `[]`. The `cmd_build` call passes `args` to `get_source_hash`.

### Incremental build failures
- Incremental build determinism correctly considers the binary hash (`binary_hash`) and the source hash to verify build outputs against tampered binaries.

### Determinism violations
- The executable bit is hashed using `st_mode & 0o111`, avoiding full `st_mode` which could include umask variability.

### Cross-platform issues
- Uses `shutil` and `os.name` checking extensively to avoid platform-specific shell utilities. Windows installs default to `APPDATA`.

