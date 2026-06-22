# SageMake

A unified Python-based build system used across the [Sage ecosystem](SPEC.md).

SageMake replaces traditional shell scripts and Makefiles with a single,
self-contained Python 3 orchestrator that handles dependency checking,
compilation, testing, installation, and cleanup. Each project in the Sage
ecosystem ships its own `sagemake` script tuned to that project's needs,
but all share a common CLI pattern, output style, and subprocess execution
model.

## Projects

| Project | Description |
|---|---|
| **SageLang** | Core language — C-bootstrapped interpreter and self-hosted compiler |
| **SageVM** | Virtual machine — compiled via SageLang, runs `.svm` bytecode |
| **SageLink** | Networking layer — encrypted tunnels over Noise_IK |
| **SageOS** | Operating system — bare-metal kernel targeting RISC-V, x64, and ARM64 |

Each project ships a `sagemake` script supporting a common set of commands:

```
./sagemake         # Full build pipeline
./sagemake build   # Build the project
./sagemake test    # Run the test suite
./sagemake install # Install to /usr/local/bin
./sagemake clean   # Remove build artifacts
```

See [SPEC.md](SPEC.md) for the complete specification.

## Making Your Own sagemake

This repo also ships an interactive tool for generating your own `sagemake` for
any Sage-based project. Run it with:

```sh
./sagemake
```

It will prompt you for your project name, description, binary name, and
dependencies, then produce a ready-to-use `sagemake` in `output/<name>/`.

You can also copy `sagemake-template` and edit the `{{ PLACEHOLDERS }}` by hand.
See [SPEC.md](SPEC.md#making-your-own-sagemake) for the full guide.
