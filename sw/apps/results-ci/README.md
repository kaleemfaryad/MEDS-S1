# M-07 Results and Automation

This document explains the generated CoreMark and Dhrystone outputs from the
M-07 parser and CI runner. Generated files belong under `build/bench/`, not in
this source directory.

`build/bench/results-history.json` records each run by default. The gate compares
CoreMark ticks per iteration and Dhrystone cycles per run with the latest
accepted result for the same configuration. These are simulated `mcycle`
deltas from functional execution under Spike, which is not cycle-accurate;
they are not actual hardware cycles or hardware performance measurements. A
regression strictly greater than 3 percent is `blocked`; an exact 3 percent
regression passes. This comparison detects changes in the Spike-run results,
not hardware performance. A blocked run is retained in history but is not used
as the next baseline.

Run both benchmarks and generate the result files from the repository root:

```bash
python3 scripts/run_ci.py --output-dir build/bench
```

The runner writes `results.json` and `results.md`, together with the benchmark
logs used to produce them. To parse existing logs directly:

```bash
python3 scripts/parse_results.py \
  --coremark path/to/coremark.log \
  --dhrystone path/to/dhrystone.log \
  --format json
```

`valid: true` means benchmark correctness/output checks passed; it does not
establish timing accuracy. Do not present CoreMark/MHz or DMIPS/MHz derived from
these Spike runs as hardware performance. Cycle-accurate performance evaluation
using Verilator is planned after PR #37, subject to the project's integration
plan.

To compare an existing result file without running the benchmarks:

```bash
python3 scripts/regression_gate.py \
  build/bench/results.json \
  --history build/bench/results-history.json \
  --record
```

`baseline` means no accepted result exists for that benchmark and configuration.
`pass` means the result is at or below the 3 percent threshold, including an
improvement. `blocked` means the result is more than 3 percent slower. Both
passing and blocked valid runs are recorded, but only passing runs become future
baselines.