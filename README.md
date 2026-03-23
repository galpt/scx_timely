# scx_timely

`scx_timely` is an experimental `sched_ext` CPU scheduler built on top of upstream [`scx_bpfland`](https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_bpfland).

Its goal is simple: adapt the TIMELY paper's delay-driven feedback idea to CPU scheduling while keeping the inherited `bpfland` base small, understandable, and close to upstream behavior.

> [!IMPORTANT]
> This project is still experimental. It should be read as a `bpfland`-based TIMELY adaptation, not as a production-ready scheduler or a literal networking-side port of the paper.

## Features

- `bpfland`-based scheduler with a narrower TIMELY-inspired control layer
- explicit Timely-style `Tlow` / `Thigh` delay regions
- queue-delay and delay-gradient feedback
- additive increase, multiplicative decrease, and HAI-style faster recovery
- built-in `desktop`, `powersave`, and `server` presets
- CLI overrides for the main Timely controller knobs
- local benchmark helpers for `mini`, `cachyos`, and `cachyos-quick`

## Modes

- `desktop`: the most validated profile so far and the main interactive preset
- `powersave`: more conservative behavior around delay growth, throttling, and recovery
- `server`: tuned around wider placement and more server-oriented policy knobs

All three modes use the same controller structure, but with different default thresholds and policy settings.

## Use Cases

`scx_timely` is aimed at people who want a scheduler that reacts to measured queue pressure instead of staying locked into one fixed policy.

Typical use cases:

- gaming and mixed desktop workloads
- low-latency creative work such as audio editing or monitoring
- development machines doing local builds while staying responsive
- heavier background work where interactive feel still matters

If you want the safest public recommendation today, the more established upstream schedulers are still the better default pick.

## TIMELY Mapping

This project follows TIMELY's control ideas, but adapts them to CPU scheduling:

- RTT -> task queue delay
- send-rate control -> per-task slice gain
- `Tlow` / `Thigh` -> low/high queue-delay thresholds
- additive increase / multiplicative decrease -> slice-gain updates
- HAI -> faster recovery after several consecutive favorable samples

So the design is TIMELY-shaped, but not a word-for-word transport-layer port.

For a short note on what changed relative to `scx_bpfland`, see [docs/design-vs-bpfland.md](docs/design-vs-bpfland.md).

## Build and Install

Build and install from source:

```bash
sudo sh install.sh --build-from-source --force
```

`install.sh` already writes `/etc/default/scx`, enables `scx.service`, and makes `scx_timely` persist across reboots.

Remove it again:

```bash
sudo sh uninstall.sh --purge --force
```

Until the next published `scx_*` crate release catches up, this repo temporarily patches the upstream `sched-ext/scx` workspace at a fixed revision so `scx_timely` stays aligned with newer inherited `bpfland` behavior.

## Benchmark Helpers

This repo ships local benchmark helpers for comparing:

- baseline Linux scheduler
- `scx_cake`
- `scx_bpfland`
- `scx_timely`

Available suites:

- `mini`
- `cachyos`
- `cachyos-quick`

Examples:

```bash
./benchmark.sh --suite mini --mode desktop
./benchmark.sh --suite mini --mode powersave
./benchmark.sh --suite mini --mode server
```

Useful helpers:

- `sudo sh enable_scx_timely.sh --flags "--mode desktop"`
- `./status_scx_timely.sh`
- `./benchmark.sh --suite mini --check-deps`
- `./benchmark.sh --suite cachyos --check-deps`
- `./benchmark.sh --suite cachyos-quick --check-deps`
- `./install_benchmark_deps.sh --mini-benchmarker --cachyos-benchmarker --plotter`
- `./install_benchmark_deps.sh --remove-workdir`
- `./kill_benchmark.sh`

If you want to re-assert `scx_timely` as the configured boot-time scheduler after changing things manually:

```bash
sudo sh enable_scx_timely.sh --flags "--mode desktop"
```

If you want a quick check of whether `scx_timely` is installed, configured in `/etc/default/scx`, and currently active:

```bash
./status_scx_timely.sh
```

For a fuller configuration check, including direct inspection of `/etc/default/scx`, run it with `sudo`.

If the benchmark helpers do not work out of the box, fetch the local scripts and plotting dependencies first:

```bash
./install_benchmark_deps.sh --mini-benchmarker --cachyos-benchmarker --plotter
```

The benchmark runner records scheduler version, exit status, and final metrics in tagged logs and generated CSV/chart output. It can also stop a run early when the scheduler has already exited, which saves time during repeated tuning.

For saved benchmark snapshots and a short note on how to interpret the adaptive scope and `exited` status, see the [`benchmark-artifacts`](https://github.com/galpt/scx_timely/tree/benchmark-artifacts) branch.

## Current Status

- `desktop`: sane enough for now and currently the best-checked preset
- `powersave`: calmer and usable enough for now, but still experimental
- `server`: first repeated `mini` runs landed in a healthy range and currently look the least problematic

These are not production-readiness claims. They are just the current local state of the profile tuning work.

## License

`scx_timely` is licensed under `GPL-2.0-only`.

## Inspirations and References

1. Mittal, R., Lam, V. T., Dukkipati, N., et al. (2015). *TIMELY: RTT-based congestion control for the datacenter.* https://research.google/pubs/timely-rtt-based-congestion-control-for-the-datacenter/
2. `sched-ext` maintainers. *scx_bpfland* [Software]. https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_bpfland
