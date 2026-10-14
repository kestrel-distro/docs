# 01 - Architecture

This document explains how Kestrel is put together and how a piece of software
travels from a source repository to a running machine.

## The model

Kestrel is an **Arch-based distribution with a desktop payload**. It is not a
from-scratch distribution and it does not mirror Arch. The base system comes
from Arch; Kestrel adds:

1. A **payload** (`kestrel-distro/kestrel`): configuration, scripts, the
   Quickshell shell, themes, the installer, and system integration.
2. A small set of **self-built packages** (a kernel and lean Hyprland and
   Quickshell) served from the `[kestrel]` repository.

Everything else is stock Arch plus the AUR, and keeps updating from Arch.

### Why a payload and not a package

The ISO copies its live root filesystem to disk (the casper/anaconda model).
There is no separate installation path and no package repository contacted
during install. The payload is therefore deployed as **files** at fixed paths,
not as a set of packages: `kestrel-apply-system-files` owns the mapping from the
payload tree to `/etc`, `/usr`, and `$PATH`. See [02 - Repositories](02-repositories.md).

## Layers of a running system

```
┌───────────────────────────────────────────────────────────────────┐
│ User applications (Chromium, Neovim, LibreOffice, ...)            │  Arch + AUR
├───────────────────────────────────────────────────────────────────┤
│ Desktop: Hyprland (compositor) + Quickshell (shell)               │  [kestrel] (lean builds)
├───────────────────────────────────────────────────────────────────┤
│ Kestrel payload: config, scripts, shell, installer, themes        │  files from the ISO
├───────────────────────────────────────────────────────────────────┤
│ Kernel: linux-kestrel (or stock linux)                            │  [kestrel] / Arch
├───────────────────────────────────────────────────────────────────┤
│ Arch base: systemd, glibc, mesa/nvidia, pipewire, ...             │  Arch
└───────────────────────────────────────────────────────────────────┘
```

## How a package reaches a machine

There are two delivery mechanisms.

### 1. At ISO build time (into the image)

`kestrel-iso/builder/build-iso.sh` builds a live rootfs with `mkarchiso`. For
each entry in `install/kestrel-source.packages` (`name=git-url`), it clones the
repository, runs the recipe at `packaging/local/PKGBUILD`, and installs the
result into the live rootfs from a build-time `file://` repository. The AUR
packages in `install/kestrel-aur.packages` are built the same way.

The result is a complete, installed-shaped system baked into the ISO.

### 2. After install (over the network)

The installed system's `pacman.conf` has an extra `[kestrel]` section pointing
at `https://kestrel-distro.github.io/packages/`. That repository is produced by
the publish workflow in `kestrel-distro/packages` and serves only Kestrel's own
packages. Because those packages use distinct names that `conflict` with the
Arch ones, `pacman -Syu` updates them from `[kestrel]` and Arch can never
overwrite them. See [05](05-package-repository.md) and [06](06-publishing.md).

## Why distinct package names

pacman has no "prefer repository X for package Y". Across all configured
repositories it installs the **highest version**; repository order only breaks
ties. Two consequences:

- If Kestrel shipped a package named `hyprland` at a lower version than Arch's,
  Arch's build would replace it on the next `pacman -Syu`.
- Pinning with `epoch` makes Kestrel always win, but then Kestrel owns every
  future update and a stale pin can break against Arch's rolling libraries.

The chosen design sidesteps both: `hyprland-kestrel` carries
`provides=(hyprland)`, `conflicts=(hyprland)`, `replaces=(hyprland)`. Arch's
`hyprland` can never install or overwrite it, and Kestrel's package updates
normally from `[kestrel]`.

## Signing and trust

Everything in `[kestrel]` is signed with a dedicated, passphrase-less
`Kestrel Packaging` RSA GPG key. The public half ships in the payload at
`default/pacman/kestrel.gpg`; the installer and a migration add it to the
pacman keyring and `--lsign` it. `[kestrel]` uses `SigLevel = Required`. See
[07 - Signing](07-signing.md).

## The ISO

The ISO is a hybrid Arch ISO (`kestrel-iso`) built with `mkarchiso`, bootable
via BIOS (syslinux) and UEFI (GRUB), using a zstd squashfs for the root. It
boots to a full Hyprland + Quickshell session; "Install Kestrel" writes the
running system to disk. The build also supports embedding an encrypted
**portable vault** ([08](08-portable-vault.md)).

## Diagram of the repositories

```
kestrel-distro/
├── kestrel        the desktop payload (config, scripts, shell, installer)
├── iso            the ISO builder (archiso profile + orchestration)
├── kernel         the linux-kestrel recipe + benchmark harness
├── hyprland       fork of hyprwm/Hyprland; lean build recipe
├── quickshell     fork of quickshell-mirror/quickshell; lean build recipe
├── packages       the [kestrel] binary repository (publish CI)
└── docs           this documentation
```

Details for each are in [02 - Repositories](02-repositories.md).
