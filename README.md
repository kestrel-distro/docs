# Kestrel documentation

Architecture, packaging, and maintainer documentation for [Kestrel](https://github.com/kestrel-distro/kestrel),
an Arch-based Linux distribution with a Hyprland + Quickshell desktop.

This repository is the reference for anyone working on Kestrel's infrastructure:
what each repository does, how the desktop and kernel are built, how the
package repository is published, and how to keep it all running.

## Start here

| Document | What it covers |
|---|---|
| [01 - Architecture](01-architecture.md) | The whole picture: repositories, the ISO model, how a package flows from source to an installed system |
| [02 - Repositories](02-repositories.md) | Every repository in the `kestrel-distro` org, its purpose and layout |
| [03 - Kernel](03-kernel.md) | The `linux-kestrel` kernel: recipe, variants, config, benchmarks, CI |
| [04 - Desktop](04-desktop.md) | The lean Hyprland and Quickshell builds |
| [05 - Package repository](05-package-repository.md) | The `[kestrel]` binary repository and what it serves |
| [06 - Publishing](06-publishing.md) | **How publishing works**, end to end, and the maintainer runbook |
| [07 - Signing](07-signing.md) | The packaging key: generation, storage, rotation |
| [08 - Portable vault](08-portable-vault.md) | The encrypted, session-writable vault baked into the ISO |
| [09 - Performance](09-performance.md) | The resource-minimization effort and how it is measured |
| [10 - Maintenance](10-maintenance.md) | Routine upkeep: bumping releases, syncing upstream, rebuilds |
| [11 - Troubleshooting](11-troubleshooting.md) | Known failures and fixes, especially in CI |
| [12 - Glossary](12-glossary.md) | Terms used throughout |

## The 30-second version

- Kestrel is built from an **Arch base** plus a **desktop payload**. There is
  no mandatory Kestrel package repository to install: the ISO copies a complete
  live system to disk.
- Kestrel builds a few packages of its own — a kernel and lean Hyprland and
  Quickshell — and serves them from a `[kestrel]` **overlay repository** on
  GitHub Pages so they keep updating after install.
- Those packages use **distinct names** (`linux-kestrel`, `hyprland-kestrel`,
  `quickshell-kestrel`) that `conflicts`/`replaces`/`provides` the Arch ones, so
  Arch can never overwrite them and updates flow only from `[kestrel]`.
- Publishing is a single GitHub Actions workflow in `kestrel-distro/packages`
  that builds each package from its source repository, signs them, and deploys
  the repository to Pages. See [06 - Publishing](06-publishing.md).

## Contributing conventions

Every Kestrel repository follows the same rules:

- Changes ship with tests where the project can test them.
- English in commits, code, and documentation.
- History is never rewritten.
