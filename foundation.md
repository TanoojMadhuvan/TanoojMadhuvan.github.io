---
layout: page
title: Foundation
permalink: /foundation/
---
This page is a static reference for the base RISC-V core — the foundation
that every research track in [/tracks](/tracks/) builds on. Full source:
[github.com/tanooj-comp-arch/foundation-core](https://github.com/tanooj-comp-arch/foundation-core).

## Assembler

A custom RISC-V assembler (`assembler.cpp`, C++17) that reads a plain-text
assembly source file and emits machine code as hex. Supports:

- R-type (`add`, `sub`, `and`, `or`, ...), I-type arithmetic, loads, and branches
- S-type stores, B-type branches, and label resolution (forward + backward)
- Register operands by numeric (`x0`–`x31`) or ABI name (`ra`, `sp`, `a0`, ...)
- Comments (`#`) and blank-line/whitespace tolerance

```
addi x1, x0, 12
addi x2, x0, 11
loop:
    beq x2, x0, done
    add x3, x3, x1
    sub x2, x2, x1
    beq x0, x0, loop
done:
    sd x3, 0(x0)
```

## Single-cycle core

A complete single-cycle RISC-V datapath in SystemVerilog (`rtl/`), validated
against a cycle-accurate C++ simulator (`single_cycle.cpp`) that executes the
same instruction stream and cross-checks architectural state.

| Module | Responsibility |
|---|---|
| `top.sv` | Top-level datapath wiring |
| `control.sv` | Main control unit — decodes opcode into control signals |
| `alu.sv` / `alu_control.sv` | Arithmetic/logic execution + ALU op decode |
| `regfile.sv` | 32×32-bit register file (`x0` hardwired to zero) |
| `immgen.sv` | Immediate generation/sign-extension per instruction format |
| `imem.sv` / `dmem.sv` | Instruction and data memory |

Every module has a standalone Verilator testbench (`tb/`) exercising it in
isolation before integration at the top level.

## Pipelined core

A 5-stage pipelined implementation (`pipeline/rtl/`) — IF, ID, EX, MEM, WB —
built on the same functional units as the single-cycle core, with pipeline
registers between every stage (`pipe_ifid.sv`, `pipe_idex.sv`, `pipe_exmem.sv`,
`pipe_memwb.sv`).

### Forwarding & hazards

- **`forwarding_unit.sv`** — resolves RAW data hazards by forwarding results
  from the EX/MEM and MEM/WB pipeline registers back into the EX stage,
  avoiding stall cycles on back-to-back dependent instructions wherever the
  hazard can be resolved combinationally instead.
- **`hazard_unit.sv`** — detects load-use hazards and control hazards
  (branches), stalling or flushing the pipeline as needed.

Validated with a dedicated pipeline testbench (`pipeline/tb/pipeline_tb.cpp`)
running representative dependency- and branch-heavy instruction sequences.

### Branch prediction

A two-level adaptive predictor (`bp_two_level.sv`, the GAg variant from Yeh
& Patt's original scheme) is wired into `pipeline_top.sv` as an early
ID-stage redirect — a single global 12-bit history register indexing a
4096-entry pattern table of 2-bit saturating counters. Two simpler schemes
(`bp_1bit.sv`, `bp_2bit.sv`) exist standalone for comparison but aren't
wired into the pipeline. Verified against a C++ oracle across 20 hand-written
test programs, on and off, before trusting any performance number from it.
Full writeup and results: [/tracks/branch-prediction](/tracks/branch-prediction/).

## Synthesis

The core synthesizes cleanly through [Yosys](https://yosyshq.net/yosys/):

- `synth.ys` — generic technology-independent synthesis
- `synth_nangate.ys` — mapped to the Nangate45 standard-cell library via
  [OpenROAD-flow-scripts](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts),
  with timing constraints in `synth/top.sdc`

First-pass PPA numbers (Nangate45, 45nm, 10ns/100MHz target): **510,014 µm²**
chip area — 95.9% of it is `dmem`, flattened into flip-flops instead of an
inferred SRAM macro. Timing is correspondingly rough: **-200.31 ns** worst
slack (this datapath, as synthesized, tops out around 4.75 MHz). Power
(OpenSTA, generic 50% toggle-rate estimate, not from a real program's
switching activity): **72.1 mW**, 73% of it `dmem` again. All three numbers
point at the same next step — a real memory macro instead of behavioral
flip-flops. Full writeup: [/tracks/ppa-analysis](/tracks/ppa-analysis/).
