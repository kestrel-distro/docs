# 07 - Signing

Kestrel signs its package database and packages with a dedicated GPG key. This
document covers the key, how it is used, and how to rotate it.

## The key

| | |
|---|---|
| Name | `Kestrel Packaging <packaging@kestrel.org>` |
| Algorithm | RSA 4096, sign-only, **passphrase-less** (for CI) |
| Fingerprint | `5F8AD0472FD8D7B6285ABCC437D052ADFB17C43C` |

The passphrase-less choice is deliberate: CI imports it non-interactively. The
security control is that it is a **dedicated** key (not the maintainer's
personal key) and that it only signs packages.

The **public** half ships in the distribution at
`kestrel/default/pacman/kestrel.gpg`. The **private** half lives only as the
`KESTREL_GPG_PRIVATE_KEY` secret on `kestrel-distro/packages`.

## How it is used

### Signing (publishing)

- `makepkg --sign` signs each built package (via `GPGKEY` in `builder`'s
  `~/.makepkg.conf`).
- `repo-add --sign --key "$GPG_KEY"` signs `kestrel.db`.
- The publish workflow imports the private key into `builder`'s keyring first.

### Trusting (installing)

`install/config/kestrel-repo.sh` runs on fresh installs and on existing installs
(via a migration):

1. `install -m644 kestrel.gpg /etc/pacman.d/kestrel.gpg`.
2. Extract the fingerprint: `gpg --with-colons --show-keys ... | awk '/^fpr:/'`.
3. `pacman-key --add /etc/pacman.d/kestrel.gpg` (if not already present).
4. `pacman-key --lsign-key <fingerprint>` (if not already signed).

`pacman-key --lsign-key` marks the key locally trusted without a web-of-trust
signature, which is what a distribution key needs.

### Verifying

`[kestrel]` uses `SigLevel = Required`, so pacman verifies both the database
and every package. To check by hand:

```sh
gpg --import kestrel/default/pacman/kestrel.gpg
curl -sO https://kestrel-distro.github.io/packages/kestrel.db
curl -sO https://kestrel-distro.github.io/packages/kestrel.db.sig
gpg --verify kestrel.db.sig kestrel.db        # -> Good signature
```

## Generating a new key

Run in an isolated keyring so nothing personal is touched:

```sh
export GNUPGHOME="$(mktemp -d)"; chmod 700 "$GNUPGHOME"
gpg --batch --pinentry-mode loopback --passphrase '' \
  --quick-gen-key "Kestrel Packaging <packaging@kestrel.org>" rsa4096 sign never
fpr="$(gpg --list-secret-keys --with-colons | awk -F: '/^fpr:/{print $10; exit}')"

# Public half -> the distribution
gpg --armor --export "$fpr" > kestrel/default/pacman/kestrel.gpg

# Private half -> the CI secret (read from a file; never echoed)
gpg --armor --export-secret-keys "$fpr" > /tmp/kestrel-packaging.asc
gh secret set KESTREL_GPG_PRIVATE_KEY --repo kestrel-distro/packages < /tmp/kestrel-packaging.asc
gh secret set KESTREL_GPG_KEY_ID --repo kestrel-distro/packages --body "$fpr"
shred -u /tmp/kestrel-packaging.asc
rm -rf "$GNUPGHOME"
```

## Rotating the key

1. Generate the new key as above (public into the payload, private into the
   secrets).
2. Publish a new repository build (the new key signs the DB and packages).
3. Ship the new public key in the payload and add a migration that runs
   `kestrel-repo.sh` again so installed systems add and locally sign the new
   key.
4. Keep the old key trusted for one release window, or re-sign and drop it.

Because the new public key replaces `kestrel.gpg` in the payload and the script
re-adds the key, no manual user action is required once the payload updates.

## Revocation

If the private key leaks, rotate immediately (above) and remove the old key from
`/etc/pacman.d/kestrel.gpg`. Until rotation completes, packages signed by the
old key remain valid on systems that still trust it; rotating the payload key is
the mechanism that stops new trust.
