# 06 - Publishing

**How the `[kestrel]` package repository is produced.** This is the document to
read first if you maintain Kestrel: it is the one automated pipeline that turns
source repositories into packages users install.

Everything is in one file:
[`kestrel-distro/packages/.github/workflows/publish.yml`](https://github.com/kestrel-distro/packages/blob/main/.github/workflows/publish.yml).

## What publishing produces

Each successful run builds every Kestrel package, signs them, and deploys a
pacman repository to GitHub Pages:

- `hyprland-kestrel` (from `kestrel-distro/hyprland`)
- `quickshell-kestrel` (from `kestrel-distro/quickshell`)
- `linux-kestrel` + `linux-kestrel-headers` (from `kestrel-distro/kernel`)

The result is served at `https://kestrel-distro.github.io/packages/`.

## When it runs

| Trigger | Purpose |
|---|---|
| `schedule: cron '17 3 * * *'` | daily rebuild (also catches Arch library bumps) |
| `workflow_dispatch` | manual run from the Actions UI or `gh workflow run` |
| `repository_dispatch: types: [upstream-release]` | fired by a fork's `sync-upstream.yml` when it merges a new upstream release |

Concurrency is serialized (`group: publish`), so runs never overlap.

## Required secrets

Set on the `kestrel-distro/packages` repository (Settings -> Secrets and
variables -> Actions):

| Secret | Meaning |
|---|---|
| `KESTREL_GPG_PRIVATE_KEY` | armored export of the passphrase-less Kestrel packaging key |
| `KESTREL_GPG_KEY_ID` | the signing key id/fingerprint |

The key is a dedicated RSA GPG key, not a personal one. See
[07 - Signing](07-signing.md) for generation and rotation.

One more secret is needed for event-driven rebuilds:

| Secret | Where | Meaning |
|---|---|---|
| `KESTREL_DISPATCH_TOKEN` | on each fork | a PAT with `repo` scope on `kestrel-distro/packages`; lets `sync-upstream.yml` fire the dispatch |

Without `KESTREL_DISPATCH_TOKEN` the forks still merge upstream; the daily cron
picks up the change within a day.

## The run, step by step

### Job `publish` (container: `archlinux:latest`, `--privileged`)

`GPG_KEY` is set from `secrets.KESTREL_GPG_KEY_ID`.

**1. Prepare build environment**

```sh
# retry() wraps a command in up to 5 attempts, 15s apart.
retry pacman -Sy --noconfirm --needed archlinux-keyring
retry pacman -Su --noconfirm --needed base-devel git sudo gnupg
useradd -m builder
echo 'builder ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/builder
printf 'GPGKEY=%s\n' "$GPG_KEY" >> /home/builder/.makepkg.conf
```

The `retry` wrapper exists because the Arch mirrors used by GitHub runners are
occasionally flaky; without it the job fails on a transient download error.

**2. Import signing key**

```sh
printf '%s\n' "$KESTREL_GPG_PRIVATE_KEY" > "$keyfile"        # 0600 temp file
install -o builder -g builder -m600 "$keyfile" /home/builder/signing.asc
sudo -u builder gpg --batch --import /home/builder/signing.asc
shred -u "$keyfile" /home/builder/signing.asc
```

The key is imported into **builder's** keyring (not root's), because both
`makepkg --sign` and `repo-add --sign` run as `builder`. The key is
passphrase-less, so no pinentry is needed. Secrets are passed through files, not
command lines, so they never appear in the process list.

**3. Build packages**

```sh
build_pkg() {
  local name="$1" url="$2" sub="$3"
  git clone --quiet "$url" /tmp/src
  chown -R builder:builder /tmp/src                 # the clone is root-owned otherwise
  ( cd "/tmp/src/$sub" && \
    sudo -u builder env -u CI -u GITHUB_RUN_ID -u GITHUB_ACTIONS \
      PKGDEST=/repo makepkg -s -f --sign --noconfirm --nocheck )
}
build_pkg hyprland   https://github.com/kestrel-distro/hyprland   packaging/local
build_pkg quickshell https://github.com/kestrel-distro/quickshell packaging/local
build_pkg kernel     https://github.com/kestrel-distro/kernel     packaging/local
```

- `-s` installs each package's `makedepends` from the Arch repos.
- `--sign` signs each built package with the key in `builder`'s keyring.
- `PKGDEST=/repo` collects the built packages.
- `env -u CI -u GITHUB_RUN_ID -u GITHUB_ACTIONS` forces the kernel PKGBUILD out
  of its "CI build" branch (which builds size-optimized/reduced-debug kernels
  meant only for quick compile checks, not for shipping).

**4. Assemble the repository**

```sh
cd /repo
sudo -u builder repo-add --sign --key "$GPG_KEY" kestrel.db.tar.gz ./*.pkg.tar.zst
pages="$GITHUB_WORKSPACE/pages"
rm -rf "$pages" && mkdir -p "$pages"
cp ./*.pkg.tar.zst ./*.pkg.tar.zst.sig "$pages/"
cp kestrel.db.tar.gz      "$pages/kestrel.db"
cp kestrel.db.tar.gz.sig  "$pages/kestrel.db.sig"
```

`repo-add` runs as `builder` so it can sign the database with the key in that
keyring. pacman fetches `<repo>.db` and `<repo>.db.sig`, but `repo-add` names
them `kestrel.db.tar.gz` and `kestrel.db.tar.gz.sig`, so they are copied to the
names pacman expects.

**5. Record tracked Arch dependency versions**

```sh
pkgs=(aquamarine hyprutils hyprlang hyprcursor hyprgraphics hyprland-guiutils
      qt6-base qt6-declarative qt6-wayland qt6-svg pipewire jemalloc cpptrace)
for p in "${pkgs[@]}"; do
  v="$(pacman -Si "$p" | awk -F': ' '/^Version/{print $2}')"
  printf '%s=%s\n' "$p" "$v"
done > "$pages/deps.txt"
```

`deps.txt` records the versions the build compiled against. A change in one of
these is the signal to rebuild (the daily cron already does; the file makes the
reason visible).

**6. Upload the Pages artifact** (`actions/upload-pages-artifact@v3`, path
`pages`).

### Job `deploy`

`actions/deploy-pages@v4` publishes the artifact to the `github-pages`
environment. The URL is `https://kestrel-distro.github.io/packages/`.

## GitHub Pages setup

The `packages` repository has Pages configured with **Source: GitHub Actions**
(`build_type=workflow`):

```sh
gh api --method POST repos/kestrel-distro/packages/pages -f build_type=workflow
```

If Pages is ever reset, re-run that.

## Signing and trust

Every database and package is signed. The matching public key ships inside the
distribution at `kestrel/default/pacman/kestrel.gpg`; `kestrel-repo.sh` adds and
locally signs it. `[kestrel]` uses `SigLevel = Required`.

To verify a run manually:

```sh
curl -sO https://kestrel-distro.github.io/packages/kestrel.db
curl -sO https://kestrel-distro.github.io/packages/kestrel.db.sig
gpg --verify kestrel.db.sig kestrel.db    # must say "Good signature"
```

Full key procedures are in [07 - Signing](07-signing.md).

## Adding a new package

1. Make the source repository build a package from `packaging/local/PKGBUILD`
   (or the repo root).
2. Add a `build_pkg <name> <git-url> packaging/local` line in the "Build
   packages" step.
3. Make it a distinct name that `conflicts`/`replaces`/`provides` any Arch
   package it shadows (see [05](05-package-repository.md)).
4. If the ISO must also ship it, add it to
   `kestrel/install/kestrel-source.packages`.
5. Run the workflow and verify the package appears in `kestrel.db`.

## Bumping a package version

- **kernel**: bump `_major`/`_minor`/`_tagrel` and the source checksum in
  `kernel/packaging/local/PKGBUILD`; update `PINS.md`.
- **hyprland**: bump `pkgver` and the release tarball `sha256sums` in
  `hyprland/packaging/local/PKGBUILD` (build the release that matches Arch's
  libraries).
- **quickshell**: usually nothing - it builds the fork tree and `pkgver()` is
  derived from `git describe`.

## Manual runs and re-runs

```sh
gh workflow run publish.yml --repo kestrel-distro/packages
gh run list --repo kestrel-distro/packages --limit 3
gh run watch <run-id> --repo kestrel-distro/packages
gh run view <run-id> --repo kestrel-distro/packages --log-failed
```

## Expected duration

On a GitHub-hosted 2-core runner a full run takes **~4 hours**, dominated by the
ThinLTO kernel build. Desktop packages are ~5-15 minutes each. If this becomes
too slow, reduce parallelism cost by splitting the kernel into its own less
frequent job, or build on a self-hosted/more-CPU runner.

## Security notes

- The private key is only ever exposed to the workflow via the
  `KESTREL_GPG_PRIVATE_KEY` secret, written to a `0600` file, used, and
  `shred`-ed. It never touches an argument vector or the environment after
  import.
- The key is a **dedicated** key, so its compromise does not touch the
  maintainer's personal identity.
- The workflow runs in a privileged container only because `mkinitcpio`/
  pacman package installation needs it.

## Failure modes

See [11 - Troubleshooting](11-troubleshooting.md) for the specific errors hit
during bring-up (root-owned clones, `llvm-ar` relative paths, missing
submodules, PCH/`-pthread`, mirror flakiness, building `main` against stable
libraries) and their fixes.
