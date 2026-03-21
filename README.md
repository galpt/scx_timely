# scx_timely

`scx_timely` is a `sched_ext` CPU scheduler bootstrapped from upstream [`scx_bpfland`](https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_bpfland).

The goal is to keep the base scheduler small and stable, then add TIMELY-inspired feedback control in measured steps instead of carrying over the complexity from earlier Cognis experiments.

## Current Status

- this repository currently starts from a renamed `scx_bpfland` scaffold
- scheduling behavior is still intentionally close to upstream `scx_bpfland`
- TIMELY-specific policy changes have not been introduced yet

## Design Direction

The intended direction is:

- preserve a BPF-first fast path and proven liveness behavior
- add a narrow control layer inspired by the TIMELY paper
- expose profile tuning such as `desktop`, `powersave`, and `server` as parameter changes rather than separate scheduler architectures

## Important Notes

> [!IMPORTANT]
> - this repository is at the bootstrap stage
> - the current code should be read as a clean starting base, not as a complete TIMELY implementation
> - future README claims should stay tied to measured behavior and local validation

## Inspirations and References

1. Mittal, R., Lam, V. T., Dukkipati, N., et al. (2015). *TIMELY: RTT-based congestion control for the datacenter.* https://research.google/pubs/timely-rtt-based-congestion-control-for-the-datacenter/
2. `sched-ext` maintainers. *scx_bpfland* [Software]. https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_bpfland
