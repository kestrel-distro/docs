# 09 - Performance

Kestrel's resource-minimization effort: what was changed, on what reasoning,
and how it is measured. The principle is that every change is judged against the
benchmark harness, not against intuition.

## Goal

Minimize RAM footprint, CPU wakeups, GPU idle draw, disk/binary size, and boot
time for the Hyprland + Quickshell desktop, without giving up function the
distribution depends on.

## Measurement first

The yardstick is the kernel repository's `bench/` harness (see
[03 - Kernel](03-kernel.md)). It has three modes:

- `cpu` - scheduler wakeup latency and throughput.
- `footprint` - kernel/module size, `/proc/meminfo` slab, boot time, module count.
- `desktop` - Hyprland/Quickshell idle CPU and RSS, idle GPU, wakeups.

A change is only promoted to a default if it wins there. `bench/compare` prints
the deltas.

## What was done

### Kernel

- Kept the CachyOS recipe, ThinLTO, x86-64-v3, `-O3`.
- **Dropped per-module BTF** (`DEBUG_INFO_BTF_MODULES`): no shipped tool consumes
  it, and it bloats every module. Core BTF is kept so `sched_ext` works.
- Kept `sched_ext` (`scx_bpfland` is the shipped default), DAMON (backs
  `kestrel-damon-reclaim`), PSI (systemd-oomd needs it) and NTSYNC.
- Added opt-in **`_trim_ksyms`** (`TRIM_UNUSED_KSYMS`) and **`_footprint`**
  (disable legacy/desktop-unused subsystems), both off by default and gated on a
  real A/B plus a boot test.

Rationale note: dropping `sched_ext`/BTF to save footprint was considered and
rejected - BTF is disk-only (non-resident), while `sched_ext` is a shipped
feature, so the trade was a loss.

### Desktop builds

- Hyprland built with ThinLTO and `-march=x86-64-v3` from the pinned release.
- Quickshell built with `Release`, jemalloc (lower QML RSS), ThinLTO, and unused
  modules (`X11`, `I3`, `SCREENCOPY`, `SERVICE_GREETD`) disabled.
- Both support PGO, measurement-gated.

### Runtime defaults

- Kestrel already ships lean Hyprland defaults: `rounding=0`,
  `shadow.enabled=false`, `blur.enabled=false`, `render.direct_scanout=2`,
  `render.new_render_scheduling=true`, logo/splash off, `vrr=2`.
- Added `QS_DROP_EXPENSIVE_FONTS=1` for the shell.
- **Rejected** `misc:disable_autoreload` / `QS_DISABLE_FILE_WATCHER`: the manual
  documents live config reload, so disabling watchers would break it.

## What was deliberately not done

- **Aggressive package trimming** (`kestrel-base.packages`): cups, bluez, docker,
  kdenlive, obs-studio and libreoffice are advertised, documented features.
  Removing them is a product decision, not a resource bug.
- **Dropping `sched_ext`/BTF/DAMON**: breaks shipped features for a disk-only
  gain.
- **Disabling config file watchers**: breaks documented live reload.

## How to run a comparison

On hardware, for each kernel or desktop build:

```
bench/run <label>
bench/compare bench/results/<a> bench/results/<b>
```

Keep power profile, background load, and settle time identical between runs.
The VM measures scheduler/memory/boot; GPU and frame-time need bare metal (the
`desktop` mode measures resource cost, which is the actual target).

## The full plan

The original working plan - every decision, rejection, and drift check - is kept
at `~/Documents/kestrel-resource-minimization-plan.md` in the maintainer's
checkout. This document is the distilled reference.
