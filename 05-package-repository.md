# 05 - Package repository

The `[kestrel]` binary repository is where Kestrel's own packages live after
install. It is served from GitHub Pages and produced by CI.

## What it serves

| Package | Source repo | Notes |
|---|---|---|
| `hyprland-kestrel` | `hyprland` | `provides/conflicts/replaces hyprland` |
| `quickshell-kestrel` | `quickshell` | `provides/conflicts/replaces quickshell` |
| `linux-kestrel` | `kernel` | installed alongside stock `linux` |
| `linux-kestrel-headers` | `kernel` | for DKMS (NVIDIA, etc.) |

## Where it is hosted

```
Server = https://kestrel-distro.github.io/packages/
```

GitHub Pages serves a static directory containing:

- `kestrel.db` - the repository database (a copy of `kestrel.db.tar.gz`).
- `kestrel.db.sig` - its GPG signature.
- `*.pkg.tar.zst` + `*.pkg.tar.zst.sig` - the signed packages.
- `deps.txt` - the Arch dependency versions the build was compiled against
  (visibility for the rebuild-on-lib-bump case).

The repository is owned by `kestrel-distro/packages`; the workflow that produces
it is documented in [06 - Publishing](06-publishing.md).

## Client configuration

Installed systems get this in `/etc/pacman.conf`, appended by
`install/config/kestrel-repo.sh` (called from the installer and from a
migration):

```
[kestrel]
SigLevel = Required
Server = https://kestrel-distro.github.io/packages/
```

The script also installs the public key from
`$KESTREL_PATH/default/pacman/kestrel.gpg` to `/etc/pacman.d/kestrel.gpg` and
runs `pacman-key --add` + `pacman-key --lsign-key`. See
[07 - Signing](07-signing.md).

The ISO builder installs the same section into the payload's
`default/pacman/pacman-{stable,rc,edge}.conf`; the post-install step copies the
chosen one to `/etc/pacman.conf`.

## How Arch and `[kestrel]` coexist

`[core] [extra] [multilib]` remain the base and update everything else. For
Kestrel's own packages:

- pacman installs the **highest version** across all configured repositories;
  repository order only breaks ties. There is no "prefer repository X for
  package Y".
- Kestrel sidesteps the resulting race by using **distinct names**. A package
  named `hyprland-kestrel` with `conflicts=(hyprland)` and `replaces=(hyprland)`
  means Arch's `hyprland` can never be installed or updated while
  `hyprland-kestrel` is present, and updates flow only from `[kestrel]`.
- `provides=(hyprland)` means anything depending on `hyprland` (for example
  `hyprshade`) is satisfied.

## Why not just ship same-named packages

- Same name with a lower version than Arch's: Arch replaces it on the next
  `pacman -Syu`.
- Same name with `epoch=1`: Kestrel always wins, but then Kestrel owns every
  future update and a stale pin can break against Arch's rolling libraries.
- Distinct names remove the version race entirely and keep the Arch base intact.

## The `[kestrel]` repository is an overlay, not a full mirror

It serves only Kestrel's own packages. It is **not** a full Arch mirror: "use
only `[kestrel]` for everything" would require mirroring or rebuilding all
~15,000 Arch packages - a CDN-scale project, not a GitHub Pages one. See
[01 - Architecture](01-architecture.md).
