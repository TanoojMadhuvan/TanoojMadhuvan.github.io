---
title: "Foundation Core"
slug: foundation-core
index: "00"
date_range: "Mar 20–Apr 24, 2026"
icon: 01-cpu
subtitle: "Base RISC-V Core"
summary: "The single-cycle and pipelined RISC-V core everything else builds on — assembler, forwarding, hazard detection."
description: "The base RISC-V core: a custom assembler, a single-cycle datapath, and a 5-stage pipelined implementation with forwarding and hazard detection, all validated against a cycle-accurate C++ simulator."
tags: [SystemVerilog, "C++", Verilator, Yosys]
tracks: [branch-prediction, verification, ppa-analysis]
---
This is the foundation every other project and research track on this site
builds on top of: a custom RISC-V assembler, a single-cycle datapath, and a
5-stage pipelined core with EX-stage forwarding and hazard detection — all
written in SystemVerilog and validated against a cycle-accurate C++
simulator and per-module Verilator testbenches.

Full technical writeup: [/foundation](/foundation/). Source:
[github.com/tanooj-comp-arch/foundation-core](https://github.com/tanooj-comp-arch/foundation-core).
