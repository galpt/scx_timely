# scx_timely

`scx_timely` is a `sched_ext` CPU scheduler bootstrapped from upstream [`scx_bpfland`](https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_bpfland).

The goal is to keep the base scheduler small and stable while adapting the TIMELY paper's feedback-driven low-latency / high-throughput idea to CPU scheduling in measured steps, without overcomplicating the scheduler's fast path.

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
- a small TIMELY-inspired control layer now measures queue delay, keeps a smoothed delay gradient, and uses explicit low/high-delay regions plus a middle-region update path to recover additively and back off multiplicatively
- controller updates are now gated on fresh enqueue-to-run delay samples and a small minimum control interval, so Timely does not keep reapplying control decisions too quickly during bursty feedback
- the current controller constants now live in userspace-owned mode config instead of being hidden as BPF literals, which makes the control loop easier to inspect, tune, and explain
- those controller constants can now also be overridden directly from the CLI, so controller calibration no longer requires editing source code for every experiment
- the current controller now also uses a less severe backoff curve and a higher minimum gain floor, so heavy pressure does not collapse slice budget as aggressively as before
- the built-in mode presets now expose explicit Timely `Tlow` / `Thigh` delay regions, which keeps the controller closer to the paper than the earlier `target / 2` simplification
- the controller now also applies additive increases in the middle region when delay is not rising, and a faster additive recovery path when delay is safely below `Tlow` and still falling
- saturated no-op increases at the gain ceiling are now ignored instead of being treated like real control updates, so the sampled controller state is less noisy under steady favorable conditions
- scheduler metrics now also show when the controller is being rate-limited by that interval and when updates are repeatedly landing at the Timely gain floor or ceiling
- a best-effort `cpu_release()` rescue path now re-enqueues tasks stranded in the local DSQ when a higher-priority class temporarily steals a CPU from `sched_ext`
- recent local benchmark runs, including the CachyOS-derived suites, still show watchdog exits under desktop RT pressure, so the current tree should be treated as an experimental scheduler and measurement harness rather than a solved production scheduler

## Design Direction

The intended direction is:

- preserve a BPF-first fast path and stay close to upstream `bpfland`'s base liveness model
- add a narrow control layer inspired by the TIMELY paper
- expose profile tuning such as `desktop`, `powersave`, and `server` as parameter changes rather than separate scheduler architectures

## Where Timely Fits

`scx_timely` is not trying to replace every `sched_ext` scheduler with one universal winner. The better way to read it is as a scheduler for people who specifically want a feedback-driven latency / throughput tradeoff instead of a more fixed scheduling policy.

Compared with other schedulers commonly listed in the [CachyOS `sched-ext` guide](https://wiki.cachyos.org/configuration/sched-ext/), Timely currently fits best as:

- an experimental choice for people who want a `bpfland`-based scheduler with a more explicit feedback controller
- a scheduler that tries to react to measured queue pressure, instead of relying only on static tiers, fixed profiles, or simpler direct-dispatch behavior
- a useful option for benchmarking and controller experimentation when you want to see how delay-targeted tuning changes scheduler behavior

It is not yet the right scheduler to recommend as a general “best for everyone” pick. If you want a more established upstream scheduler today, `scx_cake`, `scx_bpfland`, `scx_lavd`, and the other schedulers documented by CachyOS are still the safer public recommendations.

## Use Cases

If you just want the short version, `scx_timely` is aimed at people who want one scheduler that tries to react to changing pressure instead of staying locked into one fixed behavior.

It may be a reasonable fit if you care about:

- gaming, where you want a system that tries to stay responsive when bursts of work show up
- low-latency creative work such as audio editing, audio monitoring, or live content work, where responsiveness matters but background throughput still matters too
- mixed desktop workloads, such as coding while a browser, music player, chat apps, and local builds are all active
- source builds, media encoding, or other heavier work where you still want the machine to stay usable instead of feeling completely bogged down

The intended idea is not “always maximize throughput” or “always minimize latency.” It is to let a feedback controller react to measured queue pressure and try to balance the two.

That said, the current tree is still experimental. If you need the safest choice today, the more established upstream schedulers are still the better default recommendation.

## Modes

- `desktop` keeps the baseline interactive profile and enables preferred idle scanning
- the current built-in desktop tuning remains the most validated Timely profile so far
- `powersave` narrows the primary domain toward efficient cores and enables conservative throttling
- `powersave` now uses a wider Timely delay region together with more conservative `bpfland`-style policy knobs around primary domain, idle resume latency, throttling, and cpufreq
- `server` favors wider placement and enables more aggressive per-CPU / kthread-friendly tuning
- `server` keeps a tighter delay region than powersave, but changes the surrounding policy knobs toward locality and per-CPU friendliness
- all three modes set explicit Timely `Tlow` / `Thigh` thresholds for the controller
- advanced users can override the Timely controller knobs from the CLI without changing the source tree:
  - `--delay-target-us` (legacy shorthand for `Thigh`)
  - `--timely-tlow-us`
  - `--timely-thigh-us`
  - `--timely-gain-min-fp`
  - `--timely-gain-step-fp`
  - `--timely-backoff-high-fp`
  - `--timely-backoff-gradient-fp`
  - `--timely-gradient-margin-us`
  - `--timely-control-interval-us`
- delay gradient is used as an early warning signal, so multiplicative backoff can start before queue delay fully blows past `Thigh`
- when delay is safely below `Tlow` and clearly falling, the controller now uses a faster additive recovery step; when delay sits between `Tlow` and `Thigh` and is not rising, it still allows a regular additive increase instead of behaving like a one-way ratchet
- gain updates happen once per fresh queue-delay observation instead of on every subsequent dispatch, which keeps the control loop closer to a sampled-feedback design
- a small minimum control interval also prevents the controller from retuning too quickly when new delay samples arrive in a tight burst

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

If you want to tune Timely during a benchmark run without editing the source tree, pass extra scheduler flags through the runner:

```bash
./benchmark.sh --suite mini --mode desktop \
  --timely-arg --timely-control-interval-us \
  --timely-arg 750 \
  --timely-arg --timely-gain-step-fp \
  --timely-arg 16
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
> - the benchmark runner now starts with `scx_timely`, learns how many benchmark items Timely actually completed, and can cap the later variants to that same scope so repetitive tuning runs do not waste time on tests that will never show up in the comparison
> - when adaptive scope has not learned any ceiling yet, the runner now leaves the helper suites fully uncapped instead of accidentally forcing them down to a smaller test count
> - the CachyOS suite reuses a persistent workdir so repeated runs do not re-download the large benchmark assets every time
> - `cachyos-quick` reuses the same cached assets and only runs the early RT-pressure-heavy subset, so it is useful as a faster screening loop before spending time on the full `cachyos` suite
> - scheduler versions and scheduler exits are recorded in tagged logs, CSV output, and chart labels, because completed timing output alone does not guarantee that a `sched_ext` run stayed clean
> - scheduler-backed runs now stop as soon as the scheduler exits and immediately summarize the partial session instead of waiting for the rest of the benchmark script to finish
> - tagged logs now also keep the final scheduler metrics snapshot when the runtime emits one, which makes it easier to see whether Timely's delay controls, recovery path, or `cpu_release()` rescue path actually fired
> - the benchmark runner now prunes empty leftover directories from the benchmark workdir and `benchmark-results/`, while keeping the final folders that still contain logs, charts, or CSV summaries
> - benchmark metadata parsing now handles empty fields correctly, so baseline CSV/chart labels don't get shifted by blank scheduler-version or metrics lines
> - baseline runs now also wait for `sched_ext` to report no active scheduler in `root/ops`, so they do not accidentally inherit a stale scheduler name from the previous variant
> - generated charts and CSV summaries are written under `benchmark-results/`
> - this is local-machine benchmarking, not a universal scheduler claim

## Inspirations and References

1. Mittal, R., Lam, V. T., Dukkipati, N., et al. (2015). *TIMELY: RTT-based congestion control for the datacenter.* https://research.google/pubs/timely-rtt-based-congestion-control-for-the-datacenter/
2. `sched-ext` maintainers. *scx_bpfland* [Software]. https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_bpfland
