---
layout: home
title: ""
role: "Computer Architecture & Hardware Engineering Student"
# NOTE: branch accuracy / IPC / L1 cache / perf-vs-baseline below are placeholder
# numbers (not yet measured — branch-prediction/OoO tracks just started, and this
# core has no cache hierarchy yet). Swap in real benchmark results once you have them.
stats:
  - value: "97%"
    label: "branch prediction accuracy"
    icon: 06-target
  - value: "5"
    label: "stage pipeline"
    icon: 02-pipeline
  - value: "2.81"
    label: "IPC (sim avg)"
    icon: 03-gauge
  - value: "128KB"
    label: "L1 cache size"
    icon: 04-memory-grid
  - value: "1.43x"
    label: "performance vs. baseline"
    icon: 05-trending-up
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
