# M-07 CoreMark Benchmark Harness

## Overview

This directory contains the **CoreMark benchmark harness for MEDS-S1 M-07**.

The current implementation provides a reproducible RISC-V bare-metal CoreMark build and execution flow using:

* RISC-V RV64IMAC
* GCC 16.1.0
* Binutils 2.46
* Newlib 4.6.0
* Spike 1.1.1-dev
* CoreMark reference sources
* A temporary bare-metal RISC-V port

The purpose of this initial implementation is to establish a **known-good CoreMark baseline independently of the unfinished MEDS-S1 BSP, CRT0, linker script, UART driver, performance library, and RTL integration**.

Once the official MEDS-S1 software and hardware infrastructure is available, this harness can be adapted to use the actual MEDS-S1 BSP and execution environment.

---

## Directory Structure

```text
coremark/
├── Makefile
├── README.md
├── core_list_join.c
├── core_main.c
├── core_matrix.c
├── core_state.c
├── core_util.c
├── coremark.h
├── coremark.md5
└── barebones_riscv/
    ├── core_portme.c
    ├── core_portme.h
    ├── core_portme.mak
    ├── crt0.S
    └── ee_printf.c
```

### CoreMark reference sources

The following files contain the CoreMark benchmark implementation:

```text
core_list_join.c
core_main.c
core_matrix.c
core_state.c
core_util.c
coremark.h
```

These files are kept as the benchmark reference implementation and have not been modified for the MEDS-S1 port.

Target-specific functionality is implemented in `barebones_riscv/`.

---

# RISC-V Bare-Metal Port

The `barebones_riscv/` directory contains the temporary RISC-V platform port used for the initial MEDS-S1 M-07 implementation.

### `core_portme.c`

Provides the platform-specific functionality required by CoreMark, including:

* CoreMark seed values
* Timer implementation
* RISC-V `mcycle` access
* Spike UART output
* Newlib system-call stubs
* `_sbrk()` heap implementation
* `_exit()` implementation
* platform initialization/finalization

The timer uses the RISC-V `mcycle` CSR:

```c
csrr %0, mcycle
```

This allows CoreMark to measure elapsed processor cycles during execution.

### `core_portme.h`

Defines the timer conversion used by the temporary environment:

```c
#define CLOCKS_PER_SEC 1000000
#define TIMER_RES_DIVIDER 1
```

This is a **temporary timing conversion** for the current Spike-based environment. It must not be interpreted as the final MEDS-S1 processor clock frequency.

### `ee_printf.c`

CoreMark's formatted output is redirected to the Spike UART address:

```text
0x10000000
```

Characters are written through the memory-mapped UART interface:

```c
*(volatile unsigned char *)0x10000000UL
```

### `crt0.S`

A temporary bare-metal startup routine is provided because the official MEDS-S1 CRT0/BSP is not yet available.

The startup code:

1. Initializes the stack pointer.
2. Initializes the global pointer.
3. Clears the `.bss` section.
4. Calls `main`.
5. Calls `_exit`.
6. Enters a `wfi` loop.

The temporary stack is placed at:

```text
0x01010000
```

This corresponds to the top of the currently configured Spike memory region.

The official MEDS-S1 CRT0 and linker infrastructure should replace this temporary startup code during final integration.

---

# Toolchain

The benchmark was built and tested using the following toolchain:

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

The CoreMark build targets:

```text
RV64IMAC
```

with the following additional extensions:

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

Therefore the complete baseline configuration is:

```text
-O2 -march=rv64imac_zicsr_zifencei_zicbom_zicboz -mabi=lp64
```

CoreMark run-mode definitions are supplied separately through `XCFLAGS`.

For the performance run:

```text
-DPERFORMANCE_RUN=1
```

For the validation run:

```text
-DVALIDATION_RUN=1
```

The benchmark source itself is not changed to select between these modes.

---

# Spike Configuration

The current execution environment uses Spike as the RISC-V reference simulator.

The configured Spike command is:

```text
spike \
    --isa=RV64IMAC_Zicsr_Zicntr_Zifencei_Zicbom_Zicboz \
    -m0x10000:0x1000000
```

### Memory configuration

The simulated memory starts at:

```text
0x10000
```

with a size of:

```text
0x1000000
```

The temporary startup code places the stack at:

```text
0x01010000
```

### Zicntr

`Zicntr` is enabled in Spike because CoreMark uses the RISC-V `mcycle` counter.

Without the counter extension enabled in the Spike ISA configuration, execution can trap on the `mcycle` instruction.

---

# Building and Running

All commands below are executed from:

```text
sw/apps/coremark
```

First clean any previous build:

```bash
make clean PORT_DIR=barebones_riscv
```

---

## Performance Run

Build and execute the CoreMark performance configuration using 40 iterations:

```bash
make PORT_DIR=barebones_riscv \
     ITERATIONS=40 \
     XCFLAGS="-DPERFORMANCE_RUN=1" \
     load run1.log
```

The `PERFORMANCE_RUN` definition selects the CoreMark performance configuration.

---

## Validation Run

Build and execute the CoreMark validation configuration:

```bash
make clean PORT_DIR=barebones_riscv

make PORT_DIR=barebones_riscv \
     ITERATIONS=40 \
     XCFLAGS="-DVALIDATION_RUN=1" \
     load run2.log
```

The `VALIDATION_RUN` definition selects the CoreMark validation configuration.

---

# Performance Run Results

The performance run completed successfully on Spike.

Observed output:

```text
2K performance run parameters for coremark.
CoreMark Size    : 666
Total ticks      : 13162184
Total time (secs): 13
Iterations/Sec   : 3
Iterations       : 40
Compiler version : GCC 16.1.0
Compiler flags   : -O2 -march=rv64imac_zicsr_zifencei_zicbom_zicboz -mabi=lp64 -DPERFORMANCE_RUN=1
Memory location  : MEDS RAM
seedcrc           : 0xe9f5
[0]crclist        : 0xe714
[0]crcmatrix      : 0x1fd7
[0]crcstate       : 0x8e3a
[0]crcfinal       : 0x65c5
Correct operation validated. See README.md for run and reporting rules.
```

The important CoreMark correctness values are:

```text
seedcrc    = 0xe9f5
crclist    = 0xe714
crcmatrix  = 0x1fd7
crcstate   = 0x8e3a
crcfinal   = 0x65c5
```

---

# Validation Run Results

The validation run also completed successfully.

Observed output:

```text
2K validation run parameters for coremark.
CoreMark Size    : 666
Total ticks      : 13176747
Total time (secs): 13
Iterations/Sec   : 3
Iterations       : 40
Compiler version : GCC 16.1.0
Compiler flags   : -O2 -march=rv64imac_zicsr_zifencei_zicbom_zicboz -mabi=lp64 -DVALIDATION_RUN=1
Memory location  : MEDS RAM
seedcrc           : 0x18f2
[0]crclist        : 0xe3c1
[0]crcmatrix      : 0x0747
[0]crcstate       : 0x8d84
[0]crcfinal       : 0xf006
Correct operation validated. See README.md for run and reporting rules.
```

The important validation values are:

```text
seedcrc    = 0x18f2
crclist    = 0xe3c1
crcmatrix  = 0x0747
crcstate   = 0x8d84
crcfinal   = 0xf006
```

---

# Correctness Verification

Both CoreMark configurations successfully completed the benchmark's internal validation.

### Performance configuration

```text
crcfinal = 0x65c5
```

### Validation configuration

```text
crcfinal = 0xf006
```

CoreMark reported:

```text
Correct operation validated.
```

This confirms that the migrated benchmark sources and RISC-V port are functioning correctly in the current Spike environment.

---

# Reference Source Integrity

The file:

```text
coremark.md5
```

contains MD5 checksums for the six CoreMark reference source/header files:

```text
core_list_join.c
core_main.c
core_matrix.c
core_state.c
core_util.c
coremark.h
```

The verification command is:

```bash
md5sum -c coremark.md5
```

The current verification result is:

```text
core_list_join.c: OK
core_main.c: OK
core_matrix.c: OK
core_state.c: OK
core_util.c: OK
coremark.h: OK
```

This provides a simple check that the shipped CoreMark reference sources have not been unintentionally modified.

---

# Timing and Score Interpretation

The current implementation reads the RISC-V `mcycle` counter.

The measured performance run produced:

```text
Total ticks      : 13162184
Iterations       : 40
```

The approximate number of cycles per iteration is therefore:

```text
13162184 / 40 ≈ 329055 cycles/iteration
```

However, the current environment uses a temporary synthetic timer conversion:

```text
CLOCKS_PER_SEC = 1000000
```

and the execution is performed on Spike rather than the final MEDS-S1 RTL implementation.

Therefore:

**The current result must not be reported as the final MEDS-S1 CoreMark/MHz score.**

A final CoreMark/MHz result requires:

1. The actual MEDS-S1 processor clock frequency.
2. Execution on the intended MEDS-S1 hardware/RTL environment.
3. The official MEDS-S1 BSP and startup code.
4. The final MEDS-S1 memory/linker configuration.
5. The final performance measurement infrastructure.

The current Spike result is primarily a **functional and baseline execution result**.

---

# Temporary vs. Final MEDS-S1 Integration

The current implementation intentionally uses a local temporary platform because several MEDS-S1 components are still under development.

### Currently implemented for M-07

* CoreMark reference sources
* RISC-V bare-metal port
* Temporary CRT0
* Temporary linker/startup assumptions
* Spike UART output
* `mcycle` timing
* Newlib syscall stubs
* Temporary heap
* CoreMark performance configuration
* CoreMark validation configuration
* MD5 reference verification
* Reproducible Makefile-based build and execution

### MEDS-S1 components still to be integrated

The final implementation is expected to transition to the official MEDS-S1 components as they become available:

* Official MEDS-S1 CRT0
* Official linker script
* MEDS-S1 BSP
* MEDS-S1 UART/HAL
* `libs1_perf`
* Actual MEDS-S1 memory map
* S1-Core RTL
* RTL simulation/regression environment
* Actual processor clock frequency

The temporary `barebones_riscv/` port should therefore be considered a **bring-up and baseline environment**, not the final MEDS-S1 BSP.

---

# Why the Temporary Port Is Used

The goal of M-07 is to establish the benchmark and its run procedure independently from other unfinished MEDS-S1 components.

This allows CoreMark development and validation to proceed while the following infrastructure is being implemented separately:

```text
BSP
CRT0
Linker
UART
libs1_perf
RTL/Core integration
```

Once those components are available, the benchmark can be migrated from:

```text
CoreMark
    ↓
temporary bare-metal RISC-V port
    ↓
Spike
```

to the final environment:

```text
CoreMark
    ↓
MEDS-S1 BSP
    ↓
S1-Core
    ↓
MEDS-S1 RTL / target platform
```

---

# Current Limitations

The current implementation has the following limitations:

1. The official MEDS-S1 BSP is not yet used.
2. The current CRT0 is a temporary implementation.
3. The current memory configuration is specific to Spike.
4. UART output currently uses the Spike memory-mapped UART.
5. The current timer conversion is temporary.
6. `libs1_perf` is not yet integrated.
7. The current `LOAD` operation is only a placeholder command and does not program actual MEDS-S1 hardware.
8. The benchmark has not yet been executed on the final MEDS-S1 RTL/core.
9. No final CoreMark/MHz number is reported at this stage.
10. Dhrystone integration remains part of the broader M-07 scope and is not included in this CoreMark implementation.

---

# Reproducibility

A clean performance run can be reproduced with:

```bash
make clean PORT_DIR=barebones_riscv

make PORT_DIR=barebones_riscv \
     ITERATIONS=40 \
     XCFLAGS="-DPERFORMANCE_RUN=1" \
     load run1.log
```

A clean validation run can be reproduced with:

```bash
make clean PORT_DIR=barebones_riscv

make PORT_DIR=barebones_riscv \
     ITERATIONS=40 \
     XCFLAGS="-DVALIDATION_RUN=1" \
     load run2.log
```

Reference source integrity can be checked with:

```bash
md5sum -c coremark.md5
```

---

# Expected Validation Signatures

For the current toolchain and configuration, the following values should be used to detect unintended changes.

### Performance run

```text
seedcrc    : 0xe9f5
crclist    : 0xe714
crcmatrix  : 0x1fd7
crcstate   : 0x8e3a
crcfinal   : 0x65c5
```

### Validation run

```text
seedcrc    : 0x18f2
crclist    : 0xe3c1
crcmatrix  : 0x0747
crcstate   : 0x8d84
crcfinal   : 0xf006
```

Unexpected changes to these signatures should be investigated before treating a new benchmark result as valid.

---

# Next Steps

The next M-07 integration steps are:

1. Integrate the benchmark with the official MEDS-S1 BSP.
2. Replace the temporary `crt0.S` with the official MEDS-S1 startup code.
3. Use the official MEDS-S1 linker script and memory map.
4. Replace the temporary UART implementation with the official MEDS-S1 UART/HAL.
5. Integrate `libs1_perf` when available.
6. Execute CoreMark on the MEDS-S1 RTL/S1-Core.
7. Compare RTL results against the Spike baseline.
8. Confirm the actual MEDS-S1 processor clock frequency.
9. Calculate and report the final CoreMark performance according to the agreed reporting rules.
10. Add and validate the Dhrystone portion of M-07.

---

# Status

**Current status: CoreMark functional baseline complete.**

The CoreMark benchmark has been:

* migrated into `sw/apps/coremark/`
* built using GCC 16.1.0
* executed on Spike
* tested with both performance and validation configurations
* verified using CoreMark CRC signatures
* checked for reference-source integrity using MD5

The implementation is ready for review and subsequent integration with the official MEDS-S1 software and RTL infrastructure.
