# 08 - Portable vault

The portable vault lets a Kestrel ISO carry an encrypted folder that is
**read-only at rest** but appears writable during a session, with all changes
discarded on lock. It is meant for a portable workstation: keep repos, keys, or
credentials in the vault, use them anywhere, and leave no trace.

## Concept

- A folder is baked into the ISO as a **LUKS2 container** holding an ext4
  filesystem, placed at `/kestrel/vault.img` on the ISO's ISO9660 filesystem
  (outside the squashfs), unlocked with a passphrase.
- Unlocking opens a **device-mapper snapshot** of the decrypted container whose
  copy-on-write (COW) store lives in **compressed RAM** (zram) by default, or on
  a writable partition for unbounded capacity.
- Reads come from the container; writes go to the COW. Locking / rebooting /
  removing the drive discards the COW. **The container itself is never written.**

```
ISO (read-only)                    volatile, discarded on lock/reboot
┌───────────────────────┐
│ /kestrel/vault.img    │  cryptsetup open --readonly ─► /dev/mapper/kestrel-vault
│  LUKS2 -> ext4        │        │
└───────────────────────┘        └─ dm-snapshot COW ─► /dev/mapper/kestrel-vault-cow (rw)
                                     zram+zstd   (default, compressed RAM, 25% RAM)
                                     or USB part (unbounded, ephemeral key)
                                          │
                                     mount rw -> /run/kestrel/vault  ->  ~/Vault
```

Because the source is on the pendrive and only changed blocks go to the COW,
working on a large repository costs roughly your edits, not the repository.

## Building a vault in

At ISO build time:

```sh
kestrel-iso-make --vault ~/vault-contents [--vault-size 2G]
```

`kestrel-iso-make` builds the ISO, then calls `kestrel-iso-vault add`. The
passphrase is prompted (or read from `KESTREL_VAULT_PASS_FILE`); it never
appears on the command line.

Into an already-built ISO:

```sh
kestrel-iso-vault add kestrel.iso ~/vault-contents --out kestrel+vault.iso
```

This produces a **new** ISO, injects the container with `xorriso` (preserving
BIOS + UEFI boot), and, with `--sign`, re-signs it. Injection invalidates any
previous ISO signature, so always distribute the new `.sig` and hash.

Other subcommands:

- `kestrel-iso-vault create <folder> --out vault.img` - just build the container.
- `kestrel-iso-vault list <iso>` - report whether an ISO carries a vault.
- `kestrel-iso-vault remove <in.iso>` - rebuild without the vault.

## Using a vault

- `Super + Shift + V`, or **`kestrel vault unlock`**, opens the unlock dialog.
- The vault mounts at `/run/kestrel/vault` and is linked as `~/Vault`.
- **`kestrel vault lock`** unmounts it and discards the session.
- `kestrel vault status` reports `locked`/`unlocked`.

The Quickshell panel (`shell/plugins/vault/`) pipes the passphrase to the CLI
over stdin, so it never appears in the process list.

## COW backends

Configured in `/etc/kestrel/vault.conf` (from
`kestrel/etc/kestrel/vault.conf`):

| Key | Default | Meaning |
|---|---|---|
| `KESTREL_VAULT_CONTAINER` | empty | container path; empty = auto-detect (live medium, then LUKS label `KESTREL_VAULT`) |
| `KESTREL_VAULT_COW_BACKEND` | `zram` | `zram` (compressed RAM) or `disk` (writable partition) |
| `KESTREL_VAULT_COW_FRACTION` | `0.25` | zram `mem_limit` as a fraction of RAM |
| `KESTREL_VAULT_COW_DISKSIZE_FACTOR` | `2` | zram logical size = fraction x RAM x this (compression headroom) |
| `KESTREL_VAULT_DISK_COW_PART` | `/dev/disk/by-partlabel/KESTREL_VAULT_COW` | partition used by `disk` |
| `KESTREL_VAULT_MOUNTPOINT` | `/run/kestrel/vault` | where it is mounted |
| `KESTREL_VAULT_USER_LINK` | empty | convenience symlink; empty = `<invoking user home>/Vault` |

- **zram**: writes are compressed before they sit in RAM, so a text-heavy
  session holds several times the nominal size. Bounded by RAM; the `mem_limit`
  guard prevents a runaway session from exhausting memory.
- **disk**: an unbounded writable partition, wrapped in an ephemeral-key
  `dm-crypt` mapping so removal or power loss makes the leftover ciphertext
  unreadable. `kestrel-iso-flash --cow-size N` creates the partition in the USB
  free space after writing the ISO.

## Security properties

- Encrypted at rest; passphrase-only unlock; the container cannot be altered
  (`cryptsetup open --readonly`).
- Session changes live in RAM (zram) and evaporate on lock/reboot, or live in an
  ephemeral-key encrypted partition where losing the key discards them.
- The passphrase is fixed when the ISO is built; the header is read-only, so it
  cannot be changed in place - rebuild to change it.
- Working data in RAM is plaintext while unlocked (as with any unlocked
  container); locking clears it.

## Manual

The end-user write-up lives in `kestrel/manual/52-portable-vault.md`.
