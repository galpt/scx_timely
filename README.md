# scx_timely

`scx_timely` is a `sched_ext` CPU scheduler bootstrapped from upstream [`scx_bpfland`](https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_bpfland).

The goal is to keep the base scheduler small and stable, then add TIMELY-inspired feedback control in measured steps without overcomplicating the scheduler's fast path.

## Current Status

- this repository currently starts from a renamed `scx_bpfland` scaffold
- scheduling behavior is still intentionally close to upstream `scx_bpfland`
- `desktop`, `powersave`, and `server` modes are available as thin tuning presets over the inherited scheduler knobs
- a small TIMELY-inspired control layer now measures queue delay, uses a smoothed delay gradient as an early signal, and gently restores slice budget when pressure falls back down
- a best-effort `cpu_release()` rescue path now re-enqueues tasks stranded in the local DSQ when a higher-priority class temporarily steals a CPU from `sched_ext`
- recent local `cachyos` benchmark runs still show watchdog exits under desktop RT pressure, so the current tree should be treated as an experimental scheduler and measurement harness rather than a solved production scheduler

## Design Direction

The intended direction is:

- preserve a BPF-first fast path and proven liveness behavior
- add a narrow control layer inspired by the TIMELY paper
- expose profile tuning such as `desktop`, `powersave`, and `server` as parameter changes rather than separate scheduler architectures

## Modes

- `desktop` keeps the baseline interactive profile and enables preferred idle scanning
- `powersave` narrows the primary domain toward efficient cores and enables conservative throttling
- `server` favors wider placement and enables more aggressive per-CPU / kthread-friendly tuning
- all three modes also set a queue-delay target that the scheduler uses for mild TIMELY-style slice shaping
- delay gradient is now used as an early warning signal, so slice trimming can start before queue delay fully blows past the target
- when delay is low and falling, the control layer now recovers slice budget gradually instead of behaving like a one-way ratchet

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

For local scheduler comparisons, this repo ships one umbrella benchmark runner with two suites:

- `mini`: torvic9's Mini Benchmarker
- `cachyos`: the heavier CachyOS benchmark wrapper, but with local caching and cleaner script patching

The default `mini_benchmarker.sh` entrypoint is kept as a compatibility shortcut for the `mini` suite.

```bash
./benchmark.sh --suite mini --mode desktop
```

Useful helper commands:

- `./benchmark.sh --suite mini --check-deps`
- `./benchmark.sh --suite cachyos --check-deps`
- `./install_benchmark_deps.sh --mini-benchmarker --cachyos-benchmarker --plotter`
- `./install_benchmark_deps.sh --remove-workdir`

> [!IMPORTANT]
> - all reported benchmark values are elapsed time in seconds, so lower is better
> - both suites compare your baseline kernel scheduler against `scx_cake`, `scx_bpfland`, and `scx_timely`
> - the CachyOS suite reuses a persistent workdir so repeated runs do not re-download the large benchmark assets every time
> - scheduler versions and scheduler exits are now recorded in tagged logs, CSV output, and chart labels so completed benchmark output does not get mistaken for a clean run on the wrong binary
> - tagged logs now also keep the final scheduler metrics snapshot when the runtime emits one, which makes it easier to see whether Timely's delay controls, recovery path, or `cpu_release()` rescue path actually fired
> - generated charts and CSV summaries are written under `benchmark-results/`
> - this is local-machine benchmarking, not a universal scheduler claim

## Important Notes

> [!IMPORTANT]
> - this repository is still in an experimental stage
> - the current code should be read as a measured `bpfland`-based starting point with a growing TIMELY-inspired control layer, not as a complete TIMELY implementation
> - future README claims should stay tied to measured behavior and local validation
> - the install path is intentionally source-first for now; release-download automation can come later after the scheduler behavior settles

## Inspirations and References

1. Mittal, R., Lam, V. T., Dukkipati, N., et al. (2015). *TIMELY: RTT-based congestion control for the datacenter.* https://research.google/pubs/timely-rtt-based-congestion-control-for-the-datacenter/
2. `sched-ext` maintainers. *scx_bpfland* [Software]. https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_bpfland
