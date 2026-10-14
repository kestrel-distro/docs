# 04 - Desktop

Kestrel's desktop is Hyprland (compositor) and Quickshell (shell). Both are
built lean, from forks, and served by `[kestrel]` as distinct packages so Arch
cannot overwrite them.

## Hyprland - `kestrel-distro/hyprland`

A fork of [`hyprwm/Hyprland`](https://github.com/hyprwm/Hyprland).

### Building the pinned release, not `main`

The recipe builds the **pinned upstream release tarball** (`v0.56.2`), not the
fork's `main`. Tracking upstream `main` requires the matching `hyprutils`/
`hyprlang`/`aquamarine` git builds, which Kestrel does not ship, so `main` does
not compile against Arch's stable libraries. Pinning the release keeps ABI
compatibility with Arch. Bump `pkgver` to the next release that matches Arch's
libraries.

The fork's `main` still exists to carry Kestrel's source change and to track
upstream (via `sync-upstream.yml`).

### The recipe

`packaging/local/PKGBUILD`:

- Package `hyprland-kestrel`; `provides=(hyprland wayland-compositor)`,
  `conflicts=(hyprland)`, `replaces=(hyprland)`.
- clang, `-march=x86-64-v3`, ThinLTO, `-fuse-ld=mold`, `NO_HYPRPM=1`, stripped,
  no debug package.
- `-fno-lto` is removed from `CMakeLists.txt` by a `sed` in `prepare()` (upstream
  disables LTO for the sake of hyprpm plugins, which Kestrel does not build).
- `llvm-ar`/`llvm-ranlib`/`llvm-nm` are passed by absolute path (a bare name can
  be resolved relative to the source dir by CMake, breaking static-lib links).

Options:

| Variable | Default | Meaning |
|---|---|---|
| `_kestrel_march` | `x86-64-v3` | CPU target; `x86-64-v4` or `native` to retarget |
| `_kestrel_lto` | `thin` | `thin` or `none` |
| `_kestrel_pgo` | `no` | `no`, `generate`, `use` (see `packaging/PGO.md`) |
| `_kestrel_pgo_dir` | `/tmp/hyprland.profdata` | profile directory for PGO |

### Runtime dependencies

The release's dependencies match Arch's Hyprland package (cairo, aquamarine,
hyprcursor/hyprgraphics/hyprlang/hyprutils/hyprwire, libinput, xcb stack,
`xorg-xwayland`, etc.). XWayland is kept.

## Quickshell - `kestrel-distro/quickshell`

A fork of
[`quickshell-mirror/quickshell`](https://github.com/quickshell-mirror/quickshell).

`master` tracks upstream (currently `0.3.2`) plus the packaging recipe.
`packaging/local/PKGBUILD`:

- Package `quickshell-kestrel`; `provides=(quickshell)`,
  `conflicts=(quickshell)`, `replaces=(quickshell)`.
- `Release`, jemalloc, clang, `-march=x86-64-v3`, ThinLTO (absolute `llvm-ar`),
  stripped.
- `-DNO_PCH=ON`: avoids a clang precompiled-header/`-pthread` mismatch.
- Optional modules disabled: `X11`, `I3`, `SCREENCOPY`, `SERVICE_GREETD`,
  `CRASH_HANDLER`.

### Which modules are safe to disable

Verified against `kestrel/shell`: the shell imports Bluetooth, Networking,
Mpris, Notifications, Pam, Pipewire, Polkit, SystemTray, UPower, Hyprland,
Wayland (SessionLock/ToplevelManagement) and `Quickshell.Io`. It does **not**
use X11, I3, Screencopy or Greetd (screenshots go through `grim` + `slurp`), so
only those are off. Disabling anything else breaks the shell.

Options: `_kestrel_march`, `_kestrel_lto`, `_kestrel_pgo`, `_kestrel_pgo_dir`
(same as Hyprland).

## PGO

Both recipes support profile-guided optimization via `_kestrel_pgo`; the
two-stage procedure is in each fork's `packaging/PGO.md`. PGO is
**measurement-gated**: only ship a PGO package if it beats the normal build in
`bench/` on real hardware.

## Keeping the forks up to date

Each fork has `.github/workflows/sync-upstream.yml`, which daily merges upstream
into the fork's default branch and, if anything changed, dispatches a rebuild in
`packages`. The only expected divergence is:

- Hyprland: the one-line `-fno-lto` removal plus the packaging files.
- Quickshell: the packaging files.

Keep it that way so merges stay clean.

## Building locally

```
cd packaging/local
makepkg -s -f --noconfirm --nocheck
```

The ISO builds these via `install/kestrel-source.packages`; `[kestrel]` serves
them after install. Because they `conflict` with the Arch packages, installing
`hyprland-kestrel` replaces an existing `hyprland`.
