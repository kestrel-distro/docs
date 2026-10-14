# 02 - Repositories

Every repository under the `kestrel-distro` organization, and what lives in it.

## `kestrel-distro/kestrel` - the desktop payload

The distribution's own content. There is no Kestrel package; the ISO installs
Arch and AUR packages and then deploys this tree as files.

Key directories:

| Path | Contents |
|---|---|
| `bin/` | `kestrel-*` commands placed on `PATH` (the `kestrel <group> <command>` CLI) |
| `config/` | defaults copied into `~/.config/` for new users |
| `default/` | system integration: systemd units, libalpm hooks, Limine, snapper, Plymouth, SDDM, Hyprland defaults, pacman configs, the `[kestrel]` key |
| `etc/` | drop-ins Kestrel owns outright under `/etc` (copied verbatim by `kestrel-apply-system-files`) |
| `install/` | install and finalize scripts the ISO runs against a target |
| `shell/` | the Quickshell desktop (plugins hosted by `shell.qml`) |
| `themes/` | themes |
| `migrations/` | in-place fixes applied to existing installs when a default changes |
| `manual/` | the end-user manual |
| `test/` | acceptance tests |

Notable files for infrastructure work:

- `install/kestrel-base.packages` - Arch packages installed into the image.
- `install/kestrel-aur.packages` - AUR packages built at ISO time.
- `install/kestrel-source.packages` - git source packages built at ISO time.
- `install/config/kestrel-repo.sh` - registers `[kestrel]` and trusts its key.
- `default/pacman/pacman-{stable,rc,edge}.conf` - installed pacman configs.
- `default/pacman/kestrel.gpg` - the public packaging key.
- `bin/kestrel-apply-system-files` - maps the payload tree to fixed system paths.

## `kestrel-distro/iso` - the ISO builder

Builds the live ISO. `builder/build-iso.sh` runs inside an
`archlinux/archlinux` container: it pulls the archiso releng profile, layers the
Kestrel payload, builds the AUR and source packages, and calls `mkarchiso`.

| Path | Contents |
|---|---|
| `bin/kestrel-iso-make` | builds the ISO (optionally `--vault <folder>`) |
| `bin/kestrel-iso-boot` | boots the ISO in QEMU |
| `bin/kestrel-iso-sign` | detached-signs an ISO with the user's GPG key |
| `bin/kestrel-iso-release` | make + sign + hash a release |
| `bin/kestrel-iso-vault` | builds/injects a portable vault into an ISO |
| `bin/kestrel-iso-flash` | writes an ISO to a USB device (+ optional vault COW partition) |
| `builder/build-iso.sh` | the in-container build |
| `configs/` | archiso profile (profiledef, boot configs, airootfs overlay) |
| `archiso/` | vendored archiso submodule |

## `kestrel-distro/kernel` - the kernel recipe

The experimental `linux-kestrel` kernel, adapted from the CachyOS recipe. It is
installed **alongside** the stock Arch `linux` on the ISO (distinct package
name), never replacing it. See [03 - Kernel](03-kernel.md).

| Path | Contents |
|---|---|
| `packaging/local/` | `PKGBUILD` + `config` for `linux-kestrel` |
| `packaging/bore/` | `PKGBUILD` for the `linux-kestrel-bore` variant |
| `packaging/autofdo/` | documented AutoFDO build procedure |
| `bench/` | the A/B benchmark harness (`run`, `compare`) |
| `PINS.md` | pinned CachyOS inputs |
| `.github/workflows/build.yml` | builds `local` + `bore` in CI |

## `kestrel-distro/hyprland` - fork of hyprwm/Hyprland

A real GitHub fork. `main` tracks upstream plus the Kestrel changes. The lean
build recipe lives at `packaging/local/PKGBUILD` and builds the **pinned release
tarball**, not the fork tree. See [04 - Desktop](04-desktop.md).

## `kestrel-distro/quickshell` - fork of quickshell-mirror/quickshell

A real GitHub fork. `master` tracks upstream (currently 0.3.2) plus the
packaging recipe at `packaging/local/PKGBUILD`, which builds the fork tree.

## `kestrel-distro/packages` - the binary repository

Hosts the publish workflow and serves the `[kestrel]` repository from GitHub
Pages. It contains almost nothing but `.github/workflows/publish.yml` and this
README. See [05](05-package-repository.md) and [06](06-publishing.md).

## `kestrel-distro/docs` - this documentation

## Remotes and naming

- Canonical org: `kestrel-distro`.
- Historical remotes (`emiliano-go/eclipseOS*`) were moved; GitHub redirects,
  but local checkouts should point at `kestrel-distro/*`.
- The forks carry full upstream history. Keep the Kestrel diff minimal so
  upstream syncs (via `sync-upstream.yml` or a manual merge) stay clean.
