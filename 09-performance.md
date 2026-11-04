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
- Kept `sched_ext` (`scx_kestrel` is the shipped default; `scx_bpfland` remains the fallback), DAMON (backs
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

### Scheduler layer tuning (VM matrix)

`scx_kestrel` now uses short latency slices instead of the 20 ms default:
desktop 2 ms, terminal/audio 1 ms, active browser 3 ms, background browser
20 ms, with `min_exec_us`, `preempt_first` and peak-util holding on the
latency-sensitive layers. The broker also sets class `cpu.uclamp.min/max`;
UCLAMP feeds schedutil frequency through the sched_ext class aggregation, while
placement remains the BPF scheduler's job. PipeWire/WirePlumber/Jack have a
dedicated audio class and layer; games and short interactive tasks have
classification paths.

A clean 20-cell VM matrix (5 reps, background contention, cleanup enabled)
compared the tuned layers with the previous layer file, EEVDF and lavd:

| Metric (median) | Tuned Kestrel | Old layers | EEVDF | lavd |
|---|---:|---:|---:|---:|
| schbench p99 | 2588 µs | 2860 | 2588 | 2588 |
| schbench p90 | 981 µs | 971 | 793 | 553 |
| schbench RPS | 1667 | 1678 | 1638 | 1664 |
| cyclictest avg | 43 µs | 42 | 62 | 45 |
| CPU PSI some60 | 0.59% | 1.35 | 0.57 | 0.90 |

The tuned layers remove the old p99/PSI regression without losing RPS, but this
is a VM result: frame time, GPU, thermal and power are still unmeasured.

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

## Hot-path dependencies (kernel bridge)

The bridge's dependency work is limited to the compositor and browser hot path
and reuses interfaces the kernel already has:

- Mesa/DRM: the compositor raises its own GPU context priority through the
  existing `AMDGPU_CTX_*` / `I915_CONTEXT_PARAM_PRIORITY` ioctls; no kernel
  patch.
- PipeWire: realtime scheduling through rtkit, with `node.latency` tuned per
  device. The `rtirq` units are installed but disabled while `threadirqs` is
  absent from the installed kernel command line.
- jemalloc: `MALLOC_CONF=background_thread:true,thp:always` for the shell, as
  shipped in `default/hypr/envs.lua`.
- Fonts: `QS_DROP_EXPENSIVE_FONTS=1` and the lean Quickshell build keep shell
  text rendering off the hot path.

None of these add kernel surface; they drive the existing interfaces from the
payload.
