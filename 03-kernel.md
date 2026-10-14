# 03 - Kernel

Kestrel ships an experimental `linux-kestrel` kernel built from the CachyOS
recipe, installed **alongside** the stock Arch `linux` (never replacing it), for
measurement and for the extra scheduling/desktop tuning.

Repository: `kestrel-distro/kernel`.

## What it is

- Based on the CachyOS `linux-cachyos` PKGBUILD and the CachyOS kernel fork
  tarball, pinned in `PINS.md`.
- Compiled with **Clang ThinLTO**, x86-64-v3 (`_processor_opt=generic_v3`), `-O3`.
- Scheduler: `_cpusched=cachyos`, which enables `SCHED_BORE` (so BORE is the
  in-kernel default when `scx` is not active). A dedicated `bore` variant also
  exists.
- DAMON, NTSYNC, sched_ext (`SCHED_CLASS_EXT`), PSI and core BTF are kept.

## Layout

```
packaging/
  local/   PKGBUILD + config   -> linux-kestrel (default; built by the ISO + [kestrel])
  bore/    PKGBUILD            -> linux-kestrel-bore
  autofdo/ README.md           -> documented two-pass AutoFDO/BOLT build
bench/
  run      collect metrics for the running kernel (modes: cpu, footprint, desktop, all)
  compare  delta table between two runs
PINS.md    pinned CachyOS inputs (kernel source tag, patch branch, target)
```

## Build options

All options are environment overrides to `makepkg` (for example
`_footprint=yes makepkg -s -f`). Defaults are shown.

| Variable | Default | Meaning |
|---|---|---|
| `_kestrel_variant` | `kestrel` | package suffix: `linux-kestrel` |
| `_cpusched` | `cachyos` | `cachyos` (EEVDF/BORE + tweaks), `bore`, `bmq`, `eevdf`, `rt`, `rt-bore`, `hardened` |
| `_processor_opt` | `generic_v3` | `generic_v3` (x86-64-v3), `generic_v4`, `zen4`, `native` |
| `_use_llvm_lto` | `thin` | `thin`, `full`, `thin-dist`, `none` |
| `_cc_harder` | `yes` | `-O3` |
| `_cachy_config` | `yes` | apply the CachyOS `CONFIG_CACHY` set |
| `_HZ_ticks` | `1000` | timer frequency |
| `_tickrate` | `idle` | `idle`, `periodic`, `full` |
| `_preempt` | `full` | `full`, `lazy` |
| `_hugepage` | `always` | `always`, `madvise` |
| `_trim_ksyms` | `no` | `TRIM_UNUSED_KSYMS`; opt-in, breaks out-of-tree modules |
| `_footprint` | `no` | disable legacy/desktop-unused subsystems (see below) |
| `_build_debug` | `no` | also package a `-dbg` with unstripped vmlinux |
| `_build_nvidia_open` | `no` | build the open NVIDIA module into the package |
| `_build_zfs` | `no` | build ZFS into the package |
| `_autofdo`, `_propeller` | `no` | profiling-guided builds (see `packaging/autofdo/`) |
| `_localmodcfg` | `no` | `make localmodconfig` from a modprobed-db |

### The Kestrel config enforcement block

After the CachyOS defaults are applied, `prepare()` enforces Kestrel's choices:

**Kept (needed):** `SCHED_CLASS_EXT`, `CGROUP_BPF`, `DEBUG_INFO_BTF`, `PSI`
(systemd-oomd depends on it), `HIGH_RES_TIMERS`, `SYNC_FILE`, `DRM_AMD_DC`,
`DRM_FBDEV_EMULATION`, `NTSYNC`.

**Dropped:** `DEBUG_INFO_BTF_MODULES` (no shipped tool consumes per-module BTF;
saves per-module `.BTF`), `SCHED_CORE`, `SCHEDSTATS`, `VIRT_CPU_ACCOUNTING_GEN`.

**Opt-in, off by default:**

- `_trim_ksyms=yes` -> `TRIM_UNUSED_KSYMS`. Shrinks the kernel but breaks
  out-of-tree modules (`nvidia-open-dkms`); validate with those loaded first.
- `_footprint=yes` -> disables legacy DRM, hamradio, joystick/gameport, DVB and
  obscure filesystems (JFS, ReiserFS, GFS2, OCFS2, HFS, HFS+, Minix, SysV, BEFS,
  NILFS2, QNX, UFS, AFFS, ROMFS). Keeps storage, network, input, audio and the
  filesystems Kestrel uses (btrfs/ext4/xfs/f2fs/vfat/exfat/iso9660/squashfs/
  overlay/loop). Measure and boot-test on the supported hardware before shipping.

`packaging/local/config` is the CachyOS device config used verbatim; **do not
hand-edit it**. All Kestrel changes go through `scripts/config` in `prepare()`.

## The benchmark harness

`bench/run` collects metrics for the running kernel; `bench/compare` prints the
delta between two runs. Run once per kernel, compare.

```
bench/run stock          # booted into stock `linux`
# reboot into linux-kestrel
bench/run kestrel
bench/compare bench/results/<stock-dir> bench/results/<kestrel-dir>
```

Modes via `KESTREL_BENCH_MODE` (default `all`):

| Mode | Metrics | Where |
|---|---|---|
| `cpu` | schbench wakeup latency, cyclictest, stress-ng throughput | VM or bare metal |
| `footprint` | kernel/module bytes, `/proc/meminfo` slab, `systemd-analyze` boot, loaded modules | VM or bare metal |
| `desktop` | Hyprland + Quickshell CPU%/RSS, idle `nvidia-smi`, `powertop` wakeups | bare metal, live session |

`bench/compare` skips keys missing from either side, so mixing modes is safe.
Tools: `schbench`, `cyclictest` (rt-tests), `stress-ng`, `powertop` (root);
missing tools are recorded as `missing` and skipped.

## CI

`.github/workflows/build.yml` builds `packaging/local` and `packaging/bore` in
an `archlinux:latest` container (manual + push). It compiles; it does not boot
or benchmark - that is a hardware step.

## Updating the kernel

1. Bump `_major`/`_minor`/`_tagrel` and the source b2sum in the PKGBUILD to the
   next CachyOS tag; update `PINS.md`.
2. Rebase any Kestrel-only changes (the enforcement block) if the recipe moved.
3. Push; let CI compile.
4. Boot `linux-kestrel` and run `bench/` against the stock kernel before
   promoting it anywhere.

## Installing / testing locally

```
cd packaging/local
makepkg -s -f --noconfirm --nocheck     # builds linux-kestrel + -headers
sudo pacman -U linux-kestrel-*.pkg.tar.zst
```

The ISO builds this automatically via `install/kestrel-source.packages`, and
`[kestrel]` serves it after install.
