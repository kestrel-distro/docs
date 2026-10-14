# 12 - Glossary

Terms used across the Kestrel infrastructure.

- **ABI string** - the `hyprutils`/`aquamarine`/etc. versions a Hyprland build
  was compiled against; must match at runtime, which is why Hyprland is built
  from a pinned release rather than `main`.
- **archiso** - the Arch tool and profile format Kestrel's ISO uses (`mkarchiso`).
- **`[kestrel]`** - the overlay pacman repository serving Kestrel's own packages
  from GitHub Pages.
- **COW (copy-on-write)** - the writable layer over the read-only vault. A
  device-mapper *snapshot* here, backed by zram or a partition.
- **CachyOS recipe** - the `linux-cachyos` PKGBUILD/fork from which
  `linux-kestrel` is adapted.
- **distinct names** - Kestrel's package naming strategy: `hyprland-kestrel`
  etc., with `conflicts`/`replaces`/`provides` so Arch cannot overwrite them.
- **`deps.txt`** - a file published on Pages listing the Arch dependency
  versions a build compiled against; a change is a rebuild trigger.
- **dm-snapshot** - the kernel device-mapper target providing a writable
  snapshot of a read-only origin, used for the vault COW.
- **fork tree vs release tarball** - Hyprland builds the pinned release tarball
  (ABI-safe); Quickshell builds the fork tree (master == release).
- **ISO9660 root** - the outer filesystem of the ISO; `/kestrel/vault.img` lives
  here so it is readable without unpacking the squashfs and injectable post-build.
- **`linux-kestrel`** - the experimental kernel package, installed alongside
  stock `linux`.
- **mkarchiso** - the archiso build driver.
- **overlay repository** - a repository that adds a few packages on top of Arch;
  not a full mirror.
- **`packaging/local`** - the conventional path to a package's `PKGBUILD` inside
  its repository; the ISO builder looks here first.
- **PSI** - Pressure Stall Information; `systemd-oomd` keys on it, so it must
  stay enabled in the kernel.
- **PGO** - profile-guided optimization; a two-pass build, measurement-gated.
- **portable vault** - the encrypted, session-writable folder baked into the ISO.
- **`[kestrel]` publish workflow** - `.github/workflows/publish.yml` in
  `kestrel-distro/packages`; builds, signs, and deploys the repository.
- **`sched_ext` / scx** - pluggable BPF CPU schedulers; Kestrel ships with
  `scx_bpfland` as the default via `scx_loader`.
- **squashfs** - the compressed read-only root filesystem inside the ISO.
- **ThinLTO** - clang link-time optimization; used for the kernel and the
  desktop packages.
- **`_footprint` / `_trim_ksyms`** - opt-in kernel build options that shrink the
  kernel; off by default, A/B-gated.
- **zram** - compressed RAM block device; the default vault COW backing and
  Kestrel's swap.
