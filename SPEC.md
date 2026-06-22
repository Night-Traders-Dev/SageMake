# sagemake — Build System Specification

**Version:** 1.0.0
**Applies to:** SageLink, SageLang, SageVM, SageOS

---

## Overview

`sagemake` is a Python-based unified build system used across the Sage ecosystem. It replaces traditional shell scripts and Makefiles with a single, self-contained Python 3 orchestrator that handles dependency checking, compilation, testing, installation, and cleanup. Each project in the ecosystem ships its own `sagemake` script tuned to that project's needs, but all share a common CLI pattern, output style, and subprocess execution model.

The script is always invoked directly as an executable:

```sh
./sagemake <command> [arguments] [flags]
```

---

## Design Principles

- **Single-file orchestrator.** Each `sagemake` is a standalone `#!/usr/bin/env python3` script with no required package-level imports beyond `rich` (optional in SageVM, required in SageLang).
- **Rich terminal UI.** Output uses ANSI escape codes directly (SageLink) or the `rich` library (SageLang, SageVM) for colored banners, step indicators (`⚙`), and pass/fail icons (`✓` / `✗`).
- **Graceful degradation.** SageVM's `sagemake` detects whether `rich` is available at import time and falls back to plain ASCII output if not. SageLink implements its own lightweight ANSI helpers and never requires `rich`.
- **Fail-fast with informative errors.** Every step that fails calls `sys.exit(1)` after printing a red `✗` indicator. No silent failures.
- **Subprocess isolation.** All child processes are launched via `subprocess.run()` (or `subprocess.check_call()` in SageOS). Environment variables are explicitly forwarded or extended rather than inherited blindly.

---

## Common Output Primitives

All four `sagemake` scripts share a consistent visual vocabulary:

| Function | Output Role |
|---|---|
| `banner(title)` | Full-width section divider with cyan border |
| `section(title)` | Indented `→ Title` subsection label |
| `step(msg)` | Left-aligned step description, no newline (awaits result) |
| `ok()` / `step_ok(detail)` | Green `✓ done (detail)` appended to current line |
| `fail()` / `step_fail(detail)` | Red `✗ detail` appended, then `sys.exit(1)` |
| `warn()` / `step_warn(detail)` | Yellow `! detail` — non-fatal warning |

SageLang and SageVM use `rich` markup strings for these; SageLink implements them with raw ANSI escape code constants (`RESET`, `BOLD`, `RED`, `GREEN`, `CYAN`, etc.).

---

## Project-Level Specification

### SageLang

**Location:** `sagemake` in repository root  
**Requires:** Python 3, `rich`, `gcc`, `make`  
**Optional:** `cmake`, `glslc`, Vulkan SDK, GLFW, `libcurl`, OpenSSL, cuBLAS

#### Commands

| Subcommand | Description |
|---|---|
| *(default / no subcommand)* | Run full build pipeline (check deps → workspace → shaders → compile → test) |
| `all` | Same as default but also enables `--train` and `--chatbot` |
| `install` | Build then install `sage` to `/usr/local/bin` via `sudo make install` |
| `train` | Build the SL-TQ-LLM C trainer (enables `--train` flag) |
| `chatbot` | Compile chatbot backends (enables `--chatbot` flag) |
| `version [x.y.z]` | Read or propagate version across all VERSION files |

#### Flags

| Flag | Effect |
|---|---|
| `--install` | Install after build |
| `--skip-tests` | Skip test suite |
| `--make-only` | Force Make backend even if CMake is available |
| `--train` | Build the SL-TQ-LLM C trainer |
| `--chatbot` | Compile chatbot targets |
| `--no-vulkan` | Disable Vulkan GPU support |
| `--no-glfw` | Disable GLFW windowed mode |
| `--no-curl` | Disable HTTP/libcurl support |
| `--no-ssl` | Disable OpenSSL support |
| `--no-gpu` | Disable all GPU features (implies `--no-vulkan` and `--no-glfw`) |
| `--minimal` | Disable all optional features; core interpreter only |

#### Build Pipeline

1. **Dependency check** — Probes `gcc`, `make`, `cmake`, `glslc`, and `pkg-config` for Vulkan, GLFW, `libcurl`, and OpenSSL. Displays a `rich` table of results. Exits immediately if `gcc` or `make` is missing.
2. **Workspace preparation** — Runs `./sagesync` if present; otherwise runs `git submodule update --init --recursive`. Removes `core/build_sage/` and runs `make -C core clean`.
3. **Shader compilation** — If `glslc` is available and `core/examples/shaders/` exists, compiles all `.vert`, `.frag`, and `.comp` files to `.spv` SPIR-V binaries.
4. **Compilation** — If CMake is available and `--make-only` is not set, configures with `cmake -DBUILD_SAGE=ON` in `core/build_sage/` and builds with `make -j<nproc>`. Otherwise falls back to `make -C core -j<nproc>`. Produces `sage` and `sage-lsp` binaries.
5. **Binary verification** — Confirms the built `sage` binary exists and is executable.
6. **Test suite** — Runs `core/tests/run_tests.sh` (C interpreter tests, expects ~241 cases) and `make -C core test-selfhost` (self-hosted Sage tests). Can be skipped with `--skip-tests`.
7. **Installation** (optional) — Runs `sudo make -C core install`, placing `sage` at `/usr/local/bin/sage`. Restores workspace ownership to `$SUDO_USER` if run under sudo.
8. **Trainer build** (optional) — Compiles the `SL-TQ-LLM` C trainer, auto-detecting cuBLAS for GPU acceleration.

#### Version Management

`sagemake version [x.y.z]` writes the new version to `VERSION` in the repo root and propagates it using regex replacement across all tracked source files. If no version string is provided, it re-propagates the current version.

---

### SageVM

**Location:** `sagemake` in repository root  
**Requires:** Python 3, `gcc`, `make` (via SageLang build), `git`  
**Optional:** `rich`

#### Commands

| Subcommand | Description |
|---|---|
| *(default / no subcommand)* | Full pipeline: workspace → bootstrap SageLang → build SageVM |
| `build` | Alias for the default pipeline |
| `install` | Run pipeline then install binaries to `/usr/local/bin` |

#### Flags

| Flag | Effect |
|---|---|
| `--install` | Install `sagevm` to `/usr/local/bin` after build |
| `--debug` | Compile with `-DSAGE_DEBUG` flag |
| `--rebuild-sage` | Force recompilation of the SageLang host even if a binary already exists |

#### Build Pipeline

1. **Workspace preparation** — Runs `git submodule update --init --no-recurse-submodules` to synchronize `.deps/SageLang`. Removes stale `sgvm` and `sgvmc` artifacts.
2. **SageLang bootstrap** — Checks for `.deps/SageLang/core/sage`. If missing or `--rebuild-sage` is passed, compiles SageLang via `make -C .deps/SageLang/core -j<nproc>`. The `CFLAGS_EXTRA` environment variable is forwarded.
3. **SageVM compilation** — Invokes `.deps/SageLang/core/sage --compile sagevm.sage -o sagevm` with `SAGE_PATH` set to `src:src/svm:src/srvm:<lib>`. Produces a native binary.
4. **Compatibility symlinks** — Creates `sgvm → sagevm` and `sgvmc → sagevm` for backward compatibility.
5. **Installation** (optional) — Copies `sagevm` to `/usr/local/bin/sagevm` via `sudo cp` and sets `chmod 755`.

#### Key Paths

| Path | Purpose |
|---|---|
| `.deps/SageLang/` | Git submodule containing the SageLang compiler |
| `.deps/SageLang/core/sage` | SageLang host interpreter/compiler binary |
| `.deps/SageLang/core/lib` | SageLang standard library (added to `SAGE_PATH`) |
| `src/`, `src/svm/`, `src/srvm/` | SageVM source directories |
| `sagevm` | Primary compiled output binary |

---

### SageLink

**Location:** `sagemake` in repository root  
**Requires:** Python 3, `sage` (SageLang interpreter) on `PATH`

#### Commands

| Command | Description |
|---|---|
| `info` *(default)* | Display project metadata: version, source files with sizes, test files, documentation |
| `build` | Run syntax check then test suite |
| `check` | Lint all `.sage` files in `src/` and `Testing/` |
| `test` | Run the three-suite test battery |
| `run [args...]` | Launch the SageLink CLI via `sage src/cli/sagelink.sage` |
| `clean` | Remove generated files: `*.txt`, `*.svm`, `*.sgrv`, `*.sgvm`, `identity.key`, `identity.pub`, `peers.toml` |
| `all` | Run `check` then `test` |

#### Environment

SageLink's `run()` helper always extends `SAGE_PATH` to include `sagelang-lib-crypto` (resolved to an absolute path) and the current directory `.`. This ensures the crypto library is resolvable by the interpreter at runtime.

#### Test Suite

Three suites are run in sequence by `cmd_test`:

| Suite Label | File |
|---|---|
| Crypto Primitives (RFC vectors) | `Testing/test_crypto.sage` |
| Noise_IK Handshake | `Testing/test_handshake.sage` |
| Integration (CMD/FILE/SHELL) | `Testing/test_integration.sage` |

Each suite is executed as `sage <path>`. A non-zero exit code counts as a failure. stderr output from failed suites is printed. If any suite fails, the script exits with code 1.

#### Syntax Check

`cmd_check` discovers all `*.sage` files under `src/` and `Testing/` via `rglob` and runs `sage lint <file>` on each. A file counts as failed only when the lint output contains `"syntax error"` or `"Error"` in stderr. Style/warning-only output is treated as passing.

---

### SageOS

**Location:** `src/sagemake` within the repository  
**Requires:** Python 3, `make`, `dd`, `mkfs.fat`, `mcopy`, `qemu-system-*`  
**Submodules:** SageLang (at `../SageLang`), SageVM (at `../SageVM`)

#### Commands

| Command | Arguments | Description |
|---|---|---|
| `clean` | — | Remove `src/build/`, run `make clean` in SageLang core and SageVM |
| `update` | — | `git pull` then `git submodule update --init --recursive` from root |
| `sync-rootfs` | — | Create a 64 MB FAT32 disk image, copy `rootfs/` contents and a 4 MB swap stub |
| `build` | `<arch> [device]` | Check deps, compile for target arch/device, sync rootfs. `device` defaults to `qemu` |
| `run` | `<arch>` | Launch the appropriate `qemu-system-*` command for the pre-built ELF |
| `build-run` | `<arch> [device]` | Shorthand for `build` followed immediately by `run` |
| `build-vm` | — | Clean, pull, and rebuild SageVM |
| `build-sage` | — | Clean, pull, and rebuild SageLang core |
| `version` | `<x.y.z>` | Write new version string to all tracked VERSION files |
| `test-suite` | — | Run logic unit tests via `sage` and kernel integration tests via `tests/run_tests.py` |
| `benchmark` | — | Run `make -C SageLang/core benchmarks` |

#### Supported Architectures

| `arch` value | QEMU Target | Machine / CPU |
|---|---|---|
| `rv64` | `qemu-system-riscv64` | `-M virt -cpu rv64` |
| `x64` | `qemu-system-x86_64` | *(device loader mode)* |
| `arm64` | `qemu-system-aarch64` | `-M virt -cpu cortex-a57` |

All QEMU invocations use `-m 128M -serial mon:stdio -display none`.

#### Disk Image (sync-rootfs)

The `sync_rootfs()` function builds `build/sageos.img` in four stages:

1. Allocates a blank 64 MB file via `dd if=/dev/zero of=sageos.img bs=1M count=64`
2. Formats it as FAT32 with an MBR: `mkfs.fat -F 32 --mbr=y sageos.img`
3. Recursively copies `rootfs/*` into the image root: `mcopy -i sageos.img -s rootfs/* ::/`
4. Creates and copies a 4 MB zero-filled swap stub at `/swapfile` inside the image

#### Version Tracking Files

SageOS tracks versions across six files:

```
SageLang/VERSION
SageLang/core/VERSION
SageVM/VERSION
arch/arm64/VERSION
arch/rv64/VERSION
arch/x64/VERSION
```

The `version` command writes the new string to each file that exists, silently skipping absent ones.

#### Dependency Check

Before any `build` invocation, `check_dependencies()` verifies:
- The SageLang submodule is initialized (checks for `SageLang/core/Makefile`)
- Both `SageLang/core/sgvmc` and `SageLang/core/sage` exist and are up to date

If either compiler binary is missing, it automatically calls `build_sage(clean=False)` before proceeding.

#### Logic Test Suite

`test-suite` runs the following `.sage` test files through the SageLang interpreter:

- `tests/test_vfs_logic.sage`
- `tests/test_swap_logic.sage`
- `tests/test_fat32_logic.sage`
- `tests/test_pmm_logic.sage`
- `tests/test_vmm_logic.sage`
- `tests/test_virtio_logic.sage`
- `tests/test_virtqueue.sage`

After logic tests, it runs `tests/run_tests.py` for kernel integration tests if that file exists.

---

## Ecosystem Dependency Graph

```
SageOS
├── SageLang  (submodule — provides `sage` interpreter + `sgvmc` compiler)
└── SageVM    (submodule — provides `sagevm` runtime)

SageVM
└── SageLang  (submodule at .deps/SageLang — bootstrapped during build)

SageLink
└── sage      (runtime dep — must be on PATH; crypto lib via SAGE_PATH)

SageLang
└── (self-hosted — C compiler bootstraps the interpreter)
```

---

## Exit Codes

| Code | Meaning |
|---|---|
| `0` | All steps completed successfully |
| `1` | A required step failed (compilation error, missing dependency, test failure) |

All projects follow this convention. The `step_fail()` / `fail()` helpers always call `sys.exit(1)` immediately after printing the error.

---

---

## Making Your Own sagemake

You can create a custom `sagemake` for your own Sage-based project in two ways:

### Option A — Interactive Generator (Recommended)

This repository ships its own `sagemake` that acts as an interactive template generator:

```sh
git clone https://github.com/Night-Traders-Dev/SageMake.git
cd SageMake
./sagemake
```

You will be prompted for:
- **Project name** — used in banners and help text
- **Short description** — shown in the docstring and help output
- **Binary / output name** — the artifact your build produces
- **Required dependencies** — comma-separated list of tools checked before building

The generator reads `sagemake-template`, substitutes your answers, and writes a
ready-to-use `sagemake` into `output/<project-name>/sagemake`.

```sh
cd output/<project-name>
./sagemake build
```

### Option B — Manual Template

Copy `sagemake-template` from the SageMake repository into your project root
and edit the `{{ PLACEHOLDERS }}`:

| Placeholder | Description |
|---|---|
| `{{ PROJECT_NAME }}` | Display name used in banners and help |
| `{{ PROJECT_DESCRIPTION }}` | One-line description in the docstring |
| `{{ BINARY_NAME }}` | Name of the compiled binary / artifact |
| `{{ DEPENDENCIES }}` | Python list of required CLI tools (e.g. `['gcc', 'make']`) |

### Template Structure

The template provides a complete, working sagemake skeleton with:

- **ANSI output primitives** — `banner()`, `section()`, `step()`, `ok()`, `fail()`, `step_ok()`, `step_fail()`, `step_warn()`. No external dependencies required.
- **Five stub commands** — `build`, `test`, `install`, `clean`, `all`. Each prints a banner, runs its section, and provides TODO comments where you drop in your real build logic.
- **Dependency checking** — Iterates over your required tools and fails fast if any are missing.
- **CLI dispatch** — A `commands` dict mapping command names to functions, with a usage banner for invalid input.

### Convention Checklist

When customizing your generated sagemake, follow these conventions:

1. Add new commands as `cmd_<name>(args)` functions and register them in the `commands` dict.
2. Use `banner()` at the top of every command, `section()` for each phase, and `step()`/`step_ok()`/`step_fail()` for individual operations.
3. Fail fast with `step_fail()` on errors — never mask a failure.
4. Keep it as a single, standalone file. No imports beyond the standard library and `rich` (optional).
5. Name it exactly `sagemake`, make it executable (`chmod +x sagemake`), and place it in your repository root.

For the full CLI specification and output primitive reference, see [SageLang / SageVM / SageLink / SageOS](#project-level-specification) above.

## Extending sagemake

When adding a new command to any `sagemake`:

1. Define a function `cmd_<name>(args)` (SageLink style) or a standalone function called from `main()` (SageLang/SageVM/SageOS style).
2. Register it in the `commands` dict (SageLink) or add an `elif cmd == "<name>":` branch (SageOS) or extend the `argparse` subcommand handler (SageLang/SageVM).
3. Use `banner()` for the command entry point, `section()` for logical phases, and `step()` / `step_ok()` / `step_fail()` for individual operations.
4. Ensure the function exits cleanly with code `0` on success or calls `step_fail()` / `sys.exit(1)` on error.
