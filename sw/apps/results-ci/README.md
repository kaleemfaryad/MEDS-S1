# M-07 Results and Automation

This directory contains generated CoreMark and Dhrystone logs and result files
from the M-07 parser and CI runner.

Run both benchmarks and generate the result files from the repository root:

```bash
python3 sw/apps/run_ci.py --output-dir sw/apps/results-ci
```

The runner writes `results.json` and `results.md`, together with the benchmark
logs used to produce them. To parse existing logs directly:

```bash
python3 sw/apps/parse_results.py \
  --coremark path/to/coremark.log \
  --dhrystone path/to/dhrystone.log \
  --format json
```

Results use the temporary Spike platform and must not be presented as final
MEDS-S1 hardware scores.