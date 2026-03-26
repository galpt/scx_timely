# scx_timely v2 vs scx_timely v1

This document explains the line-by-line changes in v2 relative to the `scx_timely` v1 base (which itself is built on `scx_bpfland`).

For the full `scx_bpfland` code, see: https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_bpfland

## What Stays The Same

Everything from v1 stays intact:
- TIMELY delay regions (Tlow/Thigh)
- Queue-delay and delay-gradient feedback
- Slice gain control (AIMD + HAI)
- Per-task pressure mode
- All v1 locality fallback mechanisms
- Mode-based policy surface (desktop, powersave, server)

## What v2 Adds or Changes

v2 adds **pressure-aware load-balancing** via an expand/contract mode.

## Where To Look In Code

### New v2 Configuration Variables (BPF)

[`src/bpf/main.bpf.c#L87-100`](../src/bpf/main.bpf.c#L87-100)
- Lines 87-100: v2 threshold and configuration declarations

### New v2 Global State (BPF)

[`src/bpf/main.bpf.c#L196-201`](../src/bpf/main.bpf.c#L196-201)
- Lines 196-201: v2_global_pressure, v2_expand_mode, counters

### New v2 Counters (BPF)

[`src/bpf/main.bpf.c#L178`](../src/bpf/main.bpf.c#L178)
- Line 178: nr_v2_expand_mode_dispatches, nr_v2_contract_mode_dispatches

### New v2 Functions (BPF)

[`src/bpf/main.bpf.c#L829`](../src/bpf/main.bpf.c#L829)
- Line 829: `update_global_pressure()` - updates global pressure EMA and expand/contract mode
- Line 921: `is_expand_mode_active()` - returns true if in expand mode
- Line 933: `should_expand_skip_locality()` - core policy: returns true to skip locality fallback

### Modified Enqueue Logic (BPF)

[`src/bpf/main.bpf.c#L1526`](../src/bpf/main.bpf.c#L1526)
- Line 1526: Changed from `!pressure_mode_active` to `!should_expand_skip_locality(tctx)`
- Lines 1535-1539: Track expand vs contract mode dispatches

### Global Pressure Update Called (BPF)

[`src/bpf/main.bpf.c#L1758`](../src/bpf/main.bpf.c#L1758)
- Line 1758: `update_global_pressure(tctx)` called from `timely_running()`

### New v2 Rust Config (main.rs)

[`src/main.rs#L75-76`](../src/main.rs#L75-76)
- Lines 75-76: New DEFAULT_V2_EXPAND_THRESHOLD and DEFAULT_V2_CONTRACT_THRESHOLD constants

[`src/main.rs#L100-111`](../src/main.rs#L100-111)
- Lines 100-111: New v2 fields in EffectiveConfig struct (v2_locality_fallback through v2_contract_threshold)

[`src/main.rs#L141-151`](../src/main.rs#L141-151)
- Lines 141-151: New v2 fields in Desktop mode defaults (v2_locality_fallback through v2_contract_threshold)

[`src/main.rs#L186-187`](../src/main.rs#L186-187)
- Lines 186-187: v2_expand_threshold and v2_contract_threshold in Powersave mode defaults

[`src/main.rs#L222-223`](../src/main.rs#L222-223)
- Lines 222-223: v2_expand_threshold and v2_contract_threshold in Server mode defaults

[`src/main.rs#L300-305`](../src/main.rs#L300-305)
- Lines 300-305: CLI override logic for new thresholds

[`src/main.rs#L505-518`](../src/main.rs#L505-518)
- Lines 505-518: New CLI options --v2-expand-threshold and --v2-contract-threshold

[`src/main.rs#L784-785`](../src/main.rs#L784-785)
- Lines 784-785: rodata wiring for v2ExpandThreshold and v2ContractThreshold

[`src/main.rs#L707-729`](../src/main.rs#L707-729)
- Lines 707-729: Log output includes new v2 thresholds

[`src/main.rs#L1069-1071`](../src/main.rs#L1069-1071)
- Lines 1069-1071: Metrics include nr_v2_expand_mode_dispatches and nr_v2_contract_mode_dispatches

### New v2 Metrics (stats.rs)

[`src/stats.rs#L113-117`](../src/stats.rs#L113-117)
- Lines 113-117: New metric definitions

[`src/stats.rs#L125`](../src/stats.rs#L125)
- Line 125: v2exp and v2con added to summary_line format

[`src/stats.rs#L176`](../src/stats.rs#L176)
- Line 176: v2exp and v2con added to format output

[`src/stats.rs#L294-298`](../src/stats.rs#L294-298)
- Lines 294-298: v2exp and v2con added to delta calculation

## v2 Policy Summary

### Contract Mode (Locality-First)
- Default state when pressure is low
- Allows locality fallback after idle-pick miss
- Work stays close to favored CPU set

### Expand Mode (Balance-First)
- Activated when global pressure >= v2ExpandThreshold
- Skips locality fallback, dispatches directly to shared queues
- Work spreads to reduce queue delay

### Hysteresis
All modes (desktop, powersave, server) benefit from v2 expand/contract with mode-specific thresholds:
- **Desktop**: Enter expand at 75%, exit at 50%
- **Powersave**: Enter expand at 65%, exit at 40%
- **Server**: Enter expand at 80%, exit at 55%
- Prevents oscillation around the boundary
