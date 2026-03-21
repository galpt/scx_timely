# scx_timely

`scx_timely` is a `sched_ext` CPU scheduler bootstrapped from upstream [`scx_bpfland`](https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_bpfland).

The goal is to keep the base scheduler small and stable, then add TIMELY-inspired feedback control in measured steps without overcomplicating the scheduler's fast path.

## Current Status

- this repository currently starts from a renamed `scx_bpfland` scaffold
- scheduling behavior is still intentionally close to upstream `scx_bpfland`
- `desktop`, `powersave`, and `server` modes are available as thin tuning presets over the inherited scheduler knobs
- a first TIMELY-inspired signal now measures queue delay and gently reduces slice size when delay rises above a mode-specific target

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

For quick local baseline-vs-`scx_timely` comparisons, this repo also ships a small Mini Benchmarker wrapper:

```bash
./mini_benchmarker.sh --mode desktop
```

Useful helper commands:

- `./mini_benchmarker.sh --check-deps`
- `./install_benchmark_deps.sh --mini-benchmarker --plotter`
- `./install_benchmark_deps.sh --remove-workdir`

> [!IMPORTANT]
> - all reported Mini Benchmarker values are elapsed time in seconds, so lower is better
> - the helper compares your baseline kernel scheduler against `scx_timely` in the selected mode
> - generated charts and CSV summaries are written under `benchmark-results/`
> - this is local-machine benchmarking, not a universal scheduler claim

## Important Notes

> [!IMPORTANT]
> - this repository is at the bootstrap stage
> - the current code should be read as a clean starting base, not as a complete TIMELY implementation
> - future README claims should stay tied to measured behavior and local validation
> - the install path is intentionally source-first for now; release-download automation can come later after the scheduler behavior settles

## Inspirations and References

1. Mittal, R., Lam, V. T., Dukkipati, N., et al. (2015). *TIMELY: RTT-based congestion control for the datacenter.* https://research.google/pubs/timely-rtt-based-congestion-control-for-the-datacenter/
2. `sched-ext` maintainers. *scx_bpfland* [Software]. https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_bpfland
