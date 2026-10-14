# 11 - Troubleshooting

Failures encountered while bringing up the infrastructure, and their fixes.
Most are publish-workflow issues.

## Publish workflow

### `You do not have write permission for the directory $BUILDDIR`

`makepkg` runs as the `builder` user, but the cloned source tree is owned by
root.

**Fix:** `chown -R builder:builder /tmp/src` after the `git clone` (already in
the workflow).

### `/home/builder/.makepkg.conf` key import hangs or the DB is unsigned

- Importing the private key through `sudo ... bash -c` reading stdin is fragile.
  Import from a file into the `builder` keyring instead (already in the
  workflow).
- `repo-add` must run as `builder`, not root: the key is in `builder`'s keyring.
  `sudo -u builder repo-add --sign --key "$GPG_KEY" ...`.

### `llvm-ar: No such file or directory` (with a source-relative path)

CMake resolved the bare name `llvm-ar` relative to the source directory.

**Fix:** pass absolute paths:
`-DCMAKE_AR="$(command -v llvm-ar)"` (and the same for `llvm-ranlib`,
`llvm-nm`). The `llvm` package must be a makedepend.

### Hyprland: `The source directory subprojects/udis86 does not contain a CMakeLists.txt`

The `git+` source does not fetch submodules; Hyprland vendors udis86.

**Fix:** in `prepare()`, `git submodule update --init --recursive`. (Building
the release tarball instead also avoids this, since the tarball vendors it.)

### Hyprland: `error: no matching member function for call to 'emplace'`

The fork tracked upstream `main`, which needs newer `hyprutils`/`hyprlang` than
Arch's stable packages provide.

**Fix:** build the **pinned release tarball** that matches Arch's libraries, not
`main`.

### Quickshell: `POSIX thread support was disabled in precompiled file ... but is currently enabled`

A clang PCH/`-pthread` mismatch.

**Fix:** `-DNO_PCH=ON`.

### `error: failed retrieving file ... from fastly.mirror.pkgbuild.com`

Flaky Arch mirror on the runner.

**Fix:** wrap `pacman` calls in the `retry` helper (already in the workflow).

### Publish fails immediately with `pacman: no packages were upgraded`

Same mirror flakiness; the retry helper covers it.

### Pages 404 after a run

- Confirm the run succeeded (`gh run list`), not just queued/in-progress.
- Confirm Pages is set to "GitHub Actions":
  `gh api repos/kestrel-distro/packages/pages -q .build_type` should be
  `workflow`.

## `pacman -Sy` on an installed system fails

If `[kestrel]` is enabled but the Pages database does not exist yet (no
successful publish), `pacman` cannot fetch `kestrel.db` and aborts. Do not ship
an ISO with `[kestrel]` until the first publish is green, or the first `-Sy`
fails. Verify the DB is reachable:

```sh
curl -sI https://kestrel-distro.github.io/packages/kestrel.db
```

## Signature verification fails

```sh
gpg --verify kestrel.db.sig kestrel.db
```

If it fails, the public key in the payload and the signing key in CI have
diverged - they must be the same key. See [07 - Signing](07-signing.md).

## Portable vault

- **`no vault found`** - the container is not at
  `/run/archiso/bootmnt/kestrel/vault.img` and no device is labelled
  `KESTREL_VAULT`. Check the ISO actually carries it
  (`kestrel-iso-vault list <iso>`).
- **Writes fail with `ENOSPC`** - the zram COW is full; raise
  `KESTREL_VAULT_COW_FRACTION`/`_DISKSIZE_FACTOR`, or use the disk backend.
- **`files are open under ...`** on lock - close them, or `kestrel vault lock
  --force`.
- **Vault unlocked but empty** - you may have mounted the fresh overlay before
  the lower tree; check `/run/kestrel/vault-lower`.

## Kernel build issues

- **Out-of-tree modules fail to build/load after enabling `_trim_ksyms`** -
  expected; `TRIM_UNUSED_KSYMS` removes symbols those modules need. Keep it off
  unless you have validated the modules.
- **Hardware issues after `_footprint=yes`** - the trim removes driver support.
  Boot the full kernel and check which subsystem is missing before enabling it.

## Repository / git

- Old remotes (`emiliano-go/eclipseOS*`) were moved; GitHub redirects, but
  normalize to `kestrel-distro/*`.
- `kestrel-iso` requires **GPG-signed commits** (`commit.gpgsign=true`); if
  pinentry is unavailable, either sign interactively or
  `git -c commit.gpgsign=false commit` with the maintainer's awareness.
