---
layout: home
title: ""
role: "Computer Architecture & Hardware Engineering Student"
# NOTE: real, measured numbers, not placeholders.
# - accuracy/IPC/speedup: RTL-measured on the pipelined core via Verilator, 20
#   hand-written programs, oracle-verified on/off with the predictor
#   (foundation-core/BRANCH_PREDICTOR_PERFORMANCE.md). Accuracy is the
#   steady-state headline case (520-iter loop, mispred 99.8%->2.7% once the
#   GAg table trains up), not a blanket average -- the full 20-program suite
#   deliberately includes adversarial/one-shot cases that sit near a 50%
#   coin-flip floor by design, which isn't a single flattering percentage.
# - power: SysMAC's output-stationary vs. weight-stationary comparison, real
#   Yosys/Nangate45/OpenSTA synthesis (SysMAC README).
# - this core has no cache hierarchy, so there is deliberately no cache stat here.
stats:
  - value: "97.3%"
    label: "steady-state branch accuracy"
    icon: 06-target
    color: teal
  - value: "5"
    label: "stage pipeline"
    icon: 02-pipeline
    color: orange
  - value: "0.64"
    label: "IPC (measured avg)"
    icon: 03-gauge
    color: teal
  - value: "38%"
    label: "lower power (SysMAC dataflow)"
    icon: 10-fpga-gate
    color: orange
  - value: "1.19x"
    label: "faster vs. no branch predictor"
    icon: 05-trending-up
    color: teal
  - value: "45nm"
    label: "target process node"
    icon: 01-cpu
    color: orange
highlights:
  - "Designed and implemented a 5-stage pipelined RISC-V core in SystemVerilog, with EX-stage forwarding and hazard detection to resolve data hazards without full pipeline stalls wherever possible."
  - "Built a custom RISC-V assembler and cycle-accurate C++ simulator from scratch, used to validate every RTL milestone from the single-cycle core through the pipelined implementation."
  - "Synthesized the core through Yosys against the Nangate45 (45nm) standard-cell library, with an ongoing PPA (power/performance/area) analysis track digging into the results."
---
I design, model, and optimize microarchitectures at the intersection of
hardware and software. Building a RISC-V processor from the ground up —
assembler, pipelined core, and research tracks in branch prediction,
OoO execution, and PPA analysis.
