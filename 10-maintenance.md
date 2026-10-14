# 10 - Maintenance

Routine upkeep for Kestrel's infrastructure.

## Daily / automatic

- The `packages` publish workflow runs daily at 03:17 UTC and rebuilds every
  Kestrel package against the current Arch libraries, then redeploys Pages.
- Each fork's `sync-upstream.yml` runs daily, merges upstream, and (with
  `KESTREL_DISPATCH_TOKEN`) triggers a publish when something changed.

In steady state, nothing manual is required: upstream releases flow in, the
packages rebuild, and installed systems get them via `pacman -Syu`.

## When upstream releases a new version

Nothing is required for the daily path. To ship faster, the fork's sync runs
daily anyway; you can also trigger it manually:

```sh
gh workflow run sync-upstream.yml --repo kestrel-distro/hyprland
gh workflow run sync-upstream.yml --repo kestrel-distro/quickshell
```

Hyprland is special: its recipe builds a **pinned release tarball**, so a new
upstream release does not automatically get built. Bump `pkgver` and the tarball
checksum in `hyprland/packaging/local/PKGBUILD` to the release that matches
Arch's libraries, then publish. See [04 - Desktop](04-desktop.md).

## When Arch bumps a tracked library

`deps.txt` on Pages lists the versions each build compiled against. If one of
those (aquamarine, hyprutils, hyprlang, qt6-*, pipewire, jemalloc, cpptrace)
changes and a package fails to build or run, rebuild by re-running the publish
workflow; the daily cron does this automatically.

## Rotating the packaging key

See [07 - Signing](07-signing.md).

## Adding a package

1. Build it from `packaging/local/PKGBUILD` in a repo.
2. Add it to the publish workflow's "Build packages" step.
3. Give it a distinct name that conflicts/replaces/provides the Arch package.
4. Add it to `install/kestrel-source.packages` if the ISO should ship it.

## Updating the vulkan / NVIDIA paths

NVIDIA: the installer's `install/hardware/nvidia.sh` handles driver selection
(`nvidia-open-dkms` for GSP-capable cards, `nvidia-580xx-dkms` otherwise),
`modeset=1 fbdev=1`, and the initramfs `MODULES`. `default/hypr/nvidia.lua` sets
the per-session environment. Keep the kernel's out-of-tree module support in
mind when enabling `_trim_ksyms`.

## Hardware validation checklist

Some things cannot be validated in CI and must be checked on real hardware after
changes:

- [ ] The ISO boots on both BIOS and UEFI.
- [ ] The ISO's `chromium`/`zen` and Hyprland + Quickshell start.
- [ ] `pacman -Sy` succeeds on an installed system with `[kestrel]` enabled.
- [ ] Installing `hyprland-kestrel` replaces `hyprland` cleanly.
- [ ] Portable vault: unlock, edit, lock (changes gone), reboot (changes gone),
      wrong passphrase rejected.
- [ ] Kernel A/B: `bench/run` then `bench/compare` vs stock `linux`.

## Housekeeping

- Keep the Kestrel diff in the forks minimal so upstream merges stay clean.
- Upstream's own CI (Nix/Lint) also runs on the forks and will show red; disable
  or scope it if it becomes noise.
- The old standalone package repos (`hyprland-packaging`,
  `quickshell-packaging`) were deleted after the forks were created; do not
  resurrect them.
