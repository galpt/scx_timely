# scx_timely

`scx_timely` is a `sched_ext` CPU scheduler bootstrapped from upstream [`scx_bpfland`](https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_bpfland).

The goal is to keep the base scheduler small and stable, then add TIMELY-inspired feedback control in measured steps without overcomplicating the scheduler's fast path.

> [!IMPORTANT]
> - this repository is still in an experimental stage
> - the current code should be read as a measured `bpfland`-based starting point with a growing TIMELY-inspired control layer, not as a complete TIMELY implementation
> - until the next published `scx_*` crate release catches up, this repo patches the upstream `sched-ext/scx` workspace at a fixed revision to stay aligned with the latest inherited `bpfland` base behavior
> - future README claims should stay tied to measured behavior and local validation
> - the install path is intentionally source-first for now; release-download automation can come later after the scheduler behavior settles

## Current Status

- this repository starts from a renamed `scx_bpfland` scaffold, and scheduling behavior is still intentionally close to upstream `scx_bpfland`
- the current tree temporarily tracks a newer upstream `sched-ext/scx` revision for the `scx_*` helper crates so Timely stays aligned with recent `bpfland` base changes such as `SCX_ENQ_IMMED` compatibility support before the next crates.io release lands
- `desktop`, `powersave`, and `server` modes are available as thin tuning presets over the inherited scheduler knobs
- a small TIMELY-inspired control layer now measures queue delay, keeps a smoothed delay gradient, and uses a stateful low/high-delay controller to recover additively and back off multiplicatively
- controller updates are now gated on fresh enqueue-to-run delay samples, so Timely does not keep reapplying the same control decision across repeated dispatches without new feedback
- a best-effort `cpu_release()` rescue path now re-enqueues tasks stranded in the local DSQ when a higher-priority class temporarily steals a CPU from `sched_ext`
- recent local benchmark runs, including the CachyOS-derived suites, still show watchdog exits under desktop RT pressure, so the current tree should be treated as an experimental scheduler and measurement harness rather than a solved production scheduler

## Design Direction

The intended direction is:

- preserve a BPF-first fast path and stay close to upstream `bpfland`'s base liveness model
- add a narrow control layer inspired by the TIMELY paper
- expose profile tuning such as `desktop`, `powersave`, and `server` as parameter changes rather than separate scheduler architectures

## Modes

- `desktop` keeps the baseline interactive profile and enables preferred idle scanning
- `powersave` narrows the primary domain toward efficient cores and enables conservative throttling
- `server` favors wider placement and enables more aggressive per-CPU / kthread-friendly tuning
- all three modes also set a queue-delay target that the scheduler uses for the TIMELY-style control loop
- delay gradient is used as an early warning signal, so multiplicative backoff can start before queue delay fully blows past the target
- when delay is both low and clearly falling again, the controller restores slice budget additively instead of behaving like a one-way ratchet
- gain updates happen once per fresh queue-delay observation instead of on every subsequent dispatch, which keeps the control loop closer to a sampled-feedback design

## Install

`scx_timely` currently supports source-based installation via the local helper scripts:

```bash
sudo sh install.sh --build-from-source --force
```

To remove it again:

```bash
sudo sh uninstall.sh --purge --force
```

## Benchmark Helpers

For local scheduler comparisons, this repo ships one umbrella benchmark runner with three suites:

- `mini`: torvic9's Mini Benchmarker
- `cachyos`: the heavier CachyOS benchmark wrapper, but with local caching and cleaner script patching
- `cachyos-quick`: a reduced CachyOS RT-pressure screening run that keeps the same early heavy section which has historically exposed scheduler exits faster than the full run

The default `mini_benchmarker.sh` entrypoint is kept as a compatibility shortcut for the `mini` suite.

```bash
./benchmark.sh --suite mini --mode desktop
```

Useful helper commands:

- `./benchmark.sh --suite mini --check-deps`
- `./benchmark.sh --suite cachyos --check-deps`
- `./benchmark.sh --suite cachyos-quick --check-deps`
- `./install_benchmark_deps.sh --mini-benchmarker --cachyos-benchmarker --plotter`
- `./install_benchmark_deps.sh --remove-workdir`

> [!NOTE]
> - all reported benchmark values are elapsed time in seconds, so lower is better
> - all suites compare your baseline kernel scheduler against `scx_cake`, `scx_bpfland`, and `scx_timely`
> - the CachyOS suite reuses a persistent workdir so repeated runs do not re-download the large benchmark assets every time
> - `cachyos-quick` reuses the same cached assets and only runs the early RT-pressure-heavy subset, so it is useful as a faster screening loop before spending time on the full `cachyos` suite
> - scheduler versions and scheduler exits are recorded in tagged logs, CSV output, and chart labels, because completed timing output alone does not guarantee that a `sched_ext` run stayed clean
> - scheduler-backed runs now stop as soon as the scheduler exits and immediately summarize the partial session instead of waiting for the rest of the benchmark script to finish
> - tagged logs now also keep the final scheduler metrics snapshot when the runtime emits one, which makes it easier to see whether Timely's delay controls, recovery path, or `cpu_release()` rescue path actually fired
> - the benchmark runner now prunes empty leftover directories from the benchmark workdir and `benchmark-results/`, while keeping the final folders that still contain logs, charts, or CSV summaries
> - benchmark metadata parsing now handles empty fields correctly, so baseline CSV/chart labels don't get shifted by blank scheduler-version or metrics lines
> - generated charts and CSV summaries are written under `benchmark-results/`
> - this is local-machine benchmarking, not a universal scheduler claim

## Inspirations and References

1. Mittal, R., Lam, V. T., Dukkipati, N., et al. (2015). *TIMELY: RTT-based congestion control for the datacenter.* https://research.google/pubs/timely-rtt-based-congestion-control-for-the-datacenter/
2. `sched-ext` maintainers. *scx_bpfland* [Software]. https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_bpfland
