# scx_timely vs scx_bpfland

`scx_timely` is intentionally a narrow adaptation built on top of upstream `scx_bpfland`, not a rewrite from scratch.

The goal is to keep most of the inherited `bpfland` scheduling shape intact, then add a small delay-driven control layer inspired by the TIMELY paper.

## What Stays Inherited From bpfland

- the overall Rust + BPF scheduler split
- the inherited scheduler skeleton and fast path
- the general placement / dispatch model
- the mode-based policy surface (`desktop`, `powersave`, `server`)
- the upstream-friendly bias toward keeping the scheduler understandable and close to `bpfland`

In practice, `scx_timely` should be read as `bpfland` plus a focused control-law layer, not as a separate scheduler family.

## What scx_timely Adds or Changes

The main Timely-specific changes are:

- per-task queue-delay measurement
- delay-gradient tracking
- per-task control state for slice-gain updates
- explicit low/high delay thresholds (`Tlow` / `Thigh`)
- additive increase and multiplicative decrease on top of the inherited scheduling path
- HAI-style faster recovery after consecutive favorable samples
- mode defaults and CLI knobs for the Timely controller parameters

The key practical difference is that `bpfland` mostly follows its existing scheduling policy, while `scx_timely` keeps watching queue delay and adjusts scheduling aggressiveness around that signal.

## TIMELY Idea In CPU-Scheduler Terms

The original TIMELY paper is network-oriented, so `scx_timely` adapts its ideas instead of copying the transport formulas literally.

The mapping is:

- RTT -> task queue delay
- send rate -> per-task slice gain / scheduling aggressiveness
- `Tlow` / `Thigh` -> low/high queue-delay regions

That gives the controller three main regions:

- below `Tlow`: increase more confidently
- between `Tlow` and `Thigh`: react to the delay trend
- above `Thigh`: decrease more aggressively, scaled by overshoot

HAI-style recovery is used after several consecutive favorable samples, so recovery is not triggered by every isolated good sample.

## Scope

This is a TIMELY-shaped CPU-scheduler adaptation, not a literal transport-layer port.

The design tries to stay honest about that tradeoff:

- keep the `bpfland` base recognizable
- keep the Timely logic explicit
- avoid turning the scheduler into a large unrelated experiment

## Where To Look In Code

If you want the quickest review path, these are the main entry points:

- controller state and the main Timely-shaped `task_slice()` logic:
  - [`src/bpf/main.bpf.c#L744`](../src/bpf/main.bpf.c#L744) (around lines 744-891)
- explicit `Tlow` / `Thigh`, additive increase, multiplicative decrease, HAI, and gradient handling:
  - [`src/bpf/main.bpf.c#L769`](../src/bpf/main.bpf.c#L769) (around lines 769-889)
- queue-delay and gradient measurement:
  - [`src/bpf/main.bpf.c#L1241`](../src/bpf/main.bpf.c#L1241) (around lines 1241-1247)
- mode defaults and CLI-exposed Timely knobs:
  - [`src/main.rs#L105`](../src/main.rs#L105) (around lines 105-237)
  - [`src/main.rs#L314`](../src/main.rs#L314) (around lines 314-382)
- userspace -> BPF rodata wiring for the Timely parameters:
  - [`src/main.rs#L616`](../src/main.rs#L616) (around lines 616-627)
- userspace metrics and the short summary counters used by the local benchmark wrappers:
  - [`src/stats.rs#L28`](../src/stats.rs#L28) (around lines 28-43)
