# Kestrel documentation

Architecture, packaging and maintainer documentation for
[Kestrel](https://github.com/kestrel-distro/kestrel), an Arch-based Linux
distribution with a Hyprland + Quickshell desktop, foot/fish defaults and a
kernel-to-desktop scheduling bridge.

This repository is the reference for anyone working on Kestrel's infrastructure:
what each repository does, how the desktop and kernel are built, how the package
repository is published, and how to keep it all running. For the organization
landing page, see
[`kestrel-distro/.github`](https://github.com/kestrel-distro/.github).

## Start here

| Document | What it covers |
|---|---|
| [01 - Architecture](01-architecture.md) | The whole picture: repositories, the ISO model, how a package flows from source to an installed system |
| [02 - Repositories](02-repositories.md) | Every repository in the `kestrel-distro` org, its purpose and layout |
| [03 - Kernel](03-kernel.md) | `linux-kestrel`: recipe, variants, config, hint ABI, benchmarks, CI |
| [04 - Desktop](04-desktop.md) | The lean Hyprland/Quickshell builds and the matched library closure |
| [05 - Package repository](05-package-repository.md) | The `[kestrel]` binary repository and what it serves |
| [06 - Publishing](06-publishing.md) | How publishing works, end to end, and the maintainer runbook |
| [07 - Signing](07-signing.md) | The packaging key: generation, storage, rotation |
| [08 - Portable vault](08-portable-vault.md) | The encrypted, session-writable vault baked into the ISO |
| [09 - Performance](09-performance.md) | The resource-minimization effort and how it is measured |
| [10 - Maintenance](10-maintenance.md) | Routine upkeep: bumping releases, syncing upstream, rebuilds |
| [11 - Troubleshooting](11-troubleshooting.md) | Known failures and fixes, especially in CI |
| [12 - Glossary](12-glossary.md) | Terms used throughout |

## The 30-second version

- Kestrel is built from an **Arch base** plus a **desktop payload**. The ISO
  copies a complete live system to disk; there is no mandatory package
  repository to install from.
- Kestrel builds a few packages of its own — a kernel, lean Hyprland and
  Quickshell, the desktop library closure, the hint library, the broker,
  `scx_kestrel` and its loader — and serves them from a signed `[kestrel]`
  overlay on GitHub Pages so they keep updating after install.
- Those packages use **distinct names** (`linux-kestrel`, `hyprland-kestrel`,
  `quickshell-kestrel`, `scx-kestrel`, `scx-tools-kestrel`, ...) that
  `conflicts`/`replaces`/`provides` the Arch ones, so Arch can never overwrite
  them.
- The **scheduling bridge** is the distinguishing piece: `linux-kestrel` adds a
  per-task hint to `sched_attr`, `libkestrel-hint` sets it,
  `kestrel-broker` places processes in resource-class cgroups, and
  `scx_kestrel` reads the hint through `scx_bpf_task_hint()`. `scx_bpfland`
  stays the default until the bench says otherwise.
- Publishing is a single GitHub Actions workflow in `kestrel-distro/packages`
  that builds each package from its source repository, signs them, and deploys
  the repository to Pages. See [06 - Publishing](06-publishing.md).

## Repository map

| Repository | Documentation |
|---|---|
| `kestrel-distro/kestrel` | payload, commands, defaults, manual, tests |
| `kestrel-distro/iso` | live image and installer |
| `kestrel-distro/kernel` | `linux-kestrel` and the hint patch |
| `kestrel-distro/desktop` | metapackage and Hyprland library closure |
| `kestrel-distro/foot` | terminal fork with the hint |
| `kestrel-distro/hyprland` | lean compositor source |
| `kestrel-distro/quickshell` | lean shell source |
| `kestrel-distro/scx` | `scx_kestrel` |
| `kestrel-distro/scx-loader` | loader with `SupportedSched::Kestrel` |
| `kestrel-distro/qtbase` | Qt base with Wayland explicit sync |
| `kestrel-distro/packages` | signed `[kestrel]` repository |
| `kestrel-distro/.github` | org profile and community files |

## Contributing conventions

Every Kestrel repository follows the same rules:

- Changes ship with tests where the project can test them.
- English in commits, code and documentation.
- History is never rewritten.
- Everything runs in the VM; the installer writes disks.
