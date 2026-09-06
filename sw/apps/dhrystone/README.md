# M-07 Dhrystone 2.1 Benchmark Harness

## Overview

This directory contains the **Dhrystone 2.1 benchmark harness for MEDS-S1 M-07**.

The implementation provides a reproducible RISC-V bare-metal Dhrystone build and execution flow using:

* Dhrystone 2.1 reference sources
* RISC-V RV64IMAC
* GCC 16.1.0
* Binutils 2.46
* Newlib 4.6.0
* Spike 1.1.1-dev
* RISC-V `mcycle` timing
* A temporary shared bare-metal RISC-V platform

The purpose of this implementation is to establish a **known-good Dhrystone baseline independently of the unfinished MEDS-S1 BSP, CRT0, linker script, UART driver, performance library, and RTL integration**.

Once the official MEDS-S1 software and hardware infrastructure is available, this harness can be adapted to use the actual MEDS-S1 BSP and execution environment.

---

# Directory Structure

```text
dhrystone/
├── Makefile
├── README.md
├── dhry.h
├── dhry_1.c
└── dhry_2.c

../platform/barebones_riscv/
├── crt0.S
└── syscalls.c
```

The Dhrystone directory contains the benchmark sources and its Makefile.

The temporary startup and syscall layer is shared with the other M-07 benchmark implementations:

```text
sw/apps/platform/barebones_riscv/
```

---

# Dhrystone Reference Sources

The benchmark is based on **Dhrystone 2.1**.

The main benchmark sources are:

```text
dhry.h
dhry_1.c
dhry_2.c
```

The original Dhrystone benchmark uses older C language constructs, including K&R-style function definitions.

Only the changes required for the MEDS-S1 bare-metal RISC-V environment and modern GCC compatibility have been introduced.

The benchmark algorithm and workload have not been intentionally changed.

---

# Platform Integration

## `dhry_1.c`

`dhry_1.c` contains the main Dhrystone benchmark and timing logic.

For the MEDS-S1 RISC-V configuration, the source provides:

* RISC-V `mcycle` access
* Cycle-based timing
* Configurable Dhrystone iteration count
* Bare-metal-compatible output
* Newlib compatibility
* Final cycle and cycles-per-run reporting

The RISC-V cycle counter is read using:

```c
csrr %0, mcycle
```

The benchmark records the counter before and after the Dhrystone workload.

The final result is calculated as:

```text
Total cycles = End cycles - Begin cycles
```

and:

```text
Cycles per run = Total cycles / Number of runs
```

The benchmark defaults to:

```text
DHRY_RUNS = 10000
```

The number of runs can be changed through the Makefile:

```bash
make DHRY_RUNS=20000
```

## `dhry_2.c`

`dhry_2.c` contains the remaining Dhrystone procedures.

The K&R-style function definitions were retained because they are part of the original Dhrystone 2.1 source structure, with compatibility adjustments required for compilation using GCC 16.1.0.

No algorithmic changes were introduced to the benchmark procedures.

## `dhry.h`

`dhry.h` contains the original Dhrystone type definitions, constants, structures, and declarations required by the benchmark.

---

# Shared Temporary Platform

Dhrystone uses the shared temporary bare-metal platform:

```text
sw/apps/platform/barebones_riscv/
├── crt0.S
└── syscalls.c
```

## `crt0.S`

The temporary startup code provides:

1. Stack initialization.
2. Global pointer initialization.
3. `.bss` clearing.
4. C runtime initialization.
5. Application entry through `main`.
6. `_exit()` handling.
7. Final `wfi` safety loop.

The temporary stack is placed at:

```text
0x01010000
```

This corresponds to the top of the Spike memory region currently used by M-07.

## `syscalls.c`

The shared syscall layer provides minimal Newlib-compatible system calls required by the bare-metal application:

* `_write()`
* `_close()`
* `_fstat()`
* `_isatty()`
* `_lseek()`
* `_sbrk()`
* `_exit()`

Output is sent to the Spike memory-mapped UART:

```text
0x10000000
```

The heap is initialized from the linker-provided `_end` symbol and is prevented from growing beyond the temporary stack boundary.

This platform is temporary and is not the final MEDS-S1 BSP.

---

# Toolchain

The benchmark was built and tested using:

| Component | Version   |
| --------- | --------- |
| GCC       | 16.1.0    |
| Binutils  | 2.46      |
| Newlib    | 4.6.0     |
| Spike     | 1.1.1-dev |

The compiler target is:

```text
riscv64-unknown-elf
```

The compiler used by the Makefile is:

```text
riscv64-unknown-elf-gcc
```

---

# Target ISA

The Dhrystone build targets:

```text
RV64IMAC
```

with:

```text
Zicsr
Zifencei
Zicbom
Zicboz
```

The compiler architecture option is:

```text
-march=rv64imac_zicsr_zifencei_zicbom_zicboz
```

The ABI is:

```text
-mabi=lp64
```

---

# Compiler Flags

The baseline compiler optimization and architecture flags are:

```text
-O2
-march=rv64imac_zicsr_zifencei_zicbom_zicboz
-mabi=lp64
```

The complete baseline configuration is:

```text
-O2 -march=rv64imac_zicsr_zifencei_zicbom_zicboz -mabi=lp64
```

The Dhrystone build additionally defines:

```text
-DMEDS_RISCV
```

This enables the RISC-V-specific timing and bare-metal integration code.

The current Makefile also uses:

```text
-Wno-implicit-function-declaration
-Wno-old-style-definition
```

to accommodate the legacy C structure of the Dhrystone 2.1 source.

The final link uses:

```text
-nostartfiles
```

because the benchmark uses the M-07 temporary custom startup routine:

```text
sw/apps/platform/barebones_riscv/crt0.S
```

---

# Spike Configuration

The benchmark is executed using Spike with:

```text
spike \
    --isa=RV64IMAC_Zicsr_Zicntr_Zifencei_Zicbom_Zicboz \
    -m0x10000:0x1000000
```

## Memory Configuration

The simulated memory starts at:

```text
0x10000
```

with a size of:

```text
0x1000000
```

The temporary stack is located at:

```text
0x01010000
```

## Zicntr

`Zicntr` is enabled because Dhrystone reads the RISC-V `mcycle` counter.

---

# Building

All commands below are executed from:

```text
sw/apps/dhrystone
```

Clean any previous build:

```bash
make clean
```

Build the benchmark:

```bash
make
```

Alternatively:

```bash
make compile
```

The resulting executable is:

```text
dhrystone.elf
```

---

# Running Dhrystone

The benchmark can be executed with:

```bash
make run
```

This uses the default:

```text
DHRY_RUNS=10000
```

A different iteration count can be selected with:

```bash
make DHRY_RUNS=20000 run
```

---

# Benchmark Configuration

The default number of Dhrystone iterations is:

```text
10000
```

The Makefile allows this value to be changed without modifying the source:

```bash
make DHRY_RUNS=20000 run
```

The benchmark reports:

* Number of runs
* Total cycles
* Cycles per run

The primary timing result for the current environment is **cycles per Dhrystone run**.

---

# 10,000-Run Result

The benchmark was successfully executed with:

```text
DHRY_RUNS = 10000
```

The observed result was:

```text
Dhrystone Benchmark, Version 2.1 (Language: C)

Program compiled without 'register' attribute

Execution starts, 10000 runs through Dhrystone
Execution ends

Final values ... all expected values match ...

Total cycles:                                2340019
Cycles per run through Dhrystone:           234
```

The benchmark completed successfully and the expected final values matched.

The measured result was:

```text
Total cycles = 2340019
Cycles/run   = 234
```

---

# 20,000-Run Consistency Check

A second run was performed using:

```text
DHRY_RUNS = 20000
```

The observed result was:

```text
Execution starts, 20000 runs through Dhrystone
Execution ends

Total cycles:                                4680019
Cycles per run through Dhrystone:           234
```

The result remained:

```text
Cycles/run = 234
```

This provides a basic consistency check that the measured cycles scale approximately linearly with the number of Dhrystone iterations.

---

# Timing Method

The benchmark uses the RISC-V machine cycle counter:

```text
mcycle
```

The measurement sequence is conceptually:

```text
Read mcycle
    ↓
Run Dhrystone workload
    ↓
Read mcycle
    ↓
Calculate elapsed cycles
```

The resulting measurement is:

```text
Total cycles = End_Cycles - Begin_Cycles
```

and:

```text
Cycles/run = Total cycles / DHRY_RUNS
```

For the current baseline:

```text
DHRY_RUNS = 10000
Total cycles = 2340019
Cycles/run = 234
```

---

# Why Cycles/Run Is Reported

Dhrystone is traditionally reported as **Dhrystones per second (DMIPS)** or related derived metrics.

However, the current M-07 environment runs on Spike and the final MEDS-S1 processor clock frequency has not yet been established.

Therefore, reporting Dhrystones/sec at this stage would require assuming a processor frequency that is not yet representative of the final MEDS-S1 implementation.

The current harness therefore reports:

```text
Total cycles
Cycles per run
```

as the primary baseline measurements.

Once the actual processor frequency is known, Dhrystones/sec can be calculated from the measured execution cycles and the confirmed clock frequency.

---

# Dhrystone and Benchmark Comparability

Dhrystone 2.1 is an older synthetic benchmark and has several known limitations.

In particular:

* It represents a relatively small synthetic workload.
* Compiler optimization can significantly affect the generated code.
* The benchmark predates modern processor architectures and software workloads.
* Historical Dhrystone results are not always directly comparable across toolchains and systems.
* Dhrystone should therefore be treated as a secondary benchmark rather than the sole measure of processor performance.

Nevertheless, Dhrystone remains useful for M-07 as a:

* simple integer workload
* compiler/toolchain sanity check
* basic bare-metal execution test
* regression benchmark
* lightweight comparison between simulator and RTL implementations

The M-07 benchmark suite therefore includes both **CoreMark and Dhrystone** rather than relying on Dhrystone alone.

---

# Source Compatibility Changes

The Dhrystone 2.1 reference source uses legacy C constructs that are accepted differently by modern compilers.

The M-07 port includes only compatibility changes required to build and execute the benchmark with GCC 16.1.0.

These include:

* RISC-V-specific cycle-counter timing
* MEDS-RISC-V conditional code
* required standard-library includes
* explicit function declarations needed by the modern compiler
* compatibility adjustment for the `times()` declaration
* pointer formatting fixes required to avoid incorrect format/type usage
* GCC 16-compatible handling of legacy Dhrystone procedure definitions

The Dhrystone workload and benchmark algorithm were not intentionally modified.

---

# Temporary vs. Final MEDS-S1 Integration

The current Dhrystone implementation intentionally uses the temporary M-07 platform:

```text
Dhrystone
    ↓
barebones_riscv
    ↓
Spike
```

This allows the benchmark to be developed and verified before the final MEDS-S1 software and RTL infrastructure is available.

## Currently implemented

* Dhrystone 2.1 reference benchmark
* RISC-V RV64 bare-metal build
* GCC 16.1.0 support
* `mcycle` timing
* Temporary CRT0
* Temporary Newlib syscall layer
* Spike UART output
* Configurable benchmark iteration count
* 10,000-run baseline
* 20,000-run consistency check
* Cycles/run reporting
* Reproducible Makefile build/run flow

## Still to be integrated

* Official MEDS-S1 CRT0
* Official MEDS-S1 linker script
* MEDS-S1 BSP
* Official MEDS-S1 UART/HAL
* `libs1_perf`
* Actual MEDS-S1 memory map
* S1-Core RTL
* RTL simulation/regression environment
* Actual processor clock frequency

---

# Current Limitations

The current implementation has the following limitations:

1. The official MEDS-S1 BSP is not yet used.
2. The current CRT0 is temporary.
3. The current memory map is specific to Spike.
4. UART output uses the Spike memory-mapped UART.
5. The timing measurement is based on `mcycle` in Spike.
6. `libs1_perf` is not yet integrated.
7. The benchmark has not yet been executed on the final MEDS-S1 RTL/core.
8. Dhrystones/sec and DMIPS are not reported because the final processor clock frequency is not yet known.
9. The current results should be treated as a functional and performance baseline for the temporary environment, not as final MEDS-S1 processor performance.

---

# Reproducibility

From:

```text
sw/apps/dhrystone
```

run:

```bash
make clean
make
make run
```

This executes the default 10,000-run benchmark.

For a 20,000-run consistency check:

```bash
make clean
make DHRY_RUNS=20000
make DHRY_RUNS=20000 run
```

The expected baseline result is approximately:

```text
10000 runs
Total cycles: 2340019
Cycles/run:   234
```

and:

```text
20000 runs
Total cycles: 4680019
Cycles/run:   234
```

Small changes in total cycle count may occur if the execution environment or software configuration changes. The benchmark should be rebuilt and the configuration recorded whenever results are compared.

---

# Expected Baseline

The current verified M-07 Dhrystone baseline is:

| Parameter    | Value                                   |
| ------------ | --------------------------------------- |
| Benchmark    | Dhrystone 2.1                           |
| Toolchain    | GCC 16.1.0                              |
| ISA          | RV64IMAC + Zicsr/Zifencei/Zicbom/Zicboz |
| ABI          | `lp64`                                  |
| Optimization | `-O2`                                   |
| Timing       | RISC-V `mcycle`                         |
| Simulator    | Spike 1.1.1-dev                         |
| Runs         | 10,000                                  |
| Total cycles | 2,340,019                               |
| Cycles/run   | 234                                     |

Consistency check:

|   Runs | Total cycles | Cycles/run |
| -----: | -----------: | ---------: |
| 10,000 |    2,340,019 |        234 |
| 20,000 |    4,680,019 |        234 |

---

# Next Steps

The remaining Dhrystone M-07 integration steps are:

1. Integrate Dhrystone with the official MEDS-S1 BSP.
2. Replace the temporary CRT0 with the official MEDS-S1 startup code.
3. Use the official linker script and memory map.
4. Replace the temporary UART implementation with the official MEDS-S1 UART/HAL.
5. Integrate `libs1_perf` when available.
6. Execute Dhrystone on the MEDS-S1 RTL/S1-Core.
7. Compare RTL results against the Spike baseline.
8. Confirm the actual MEDS-S1 processor clock frequency.
9. Calculate Dhrystones/sec using the confirmed processor frequency.
10. Record the final M-07 Dhrystone result according to the agreed reporting rules.

---

# Status

**Current status: Dhrystone functional baseline complete.**

Dhrystone 2.1 has been:

* integrated into `sw/apps/dhrystone/`
* built using GCC 16.1.0
* executed successfully on Spike
* tested with 10,000 iterations
* tested with 20,000 iterations
* measured using the RISC-V `mcycle` counter
* verified with matching expected final values
* checked for stable cycles-per-run behavior
* integrated with the shared temporary M-07 bare-metal platform

The verified baseline is:

```text
Dhrystone 2.1
Runs:          10000
Total cycles:  2340019
Cycles/run:    234
```

The Dhrystone implementation is ready for review and subsequent integration with the official MEDS-S1 software and RTL infrastructure.
