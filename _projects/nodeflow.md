---
title: "NodeFlow (RV32IM)"
slug: nodeflow
index: "03"
date_range: "Jul 25, 2026–ongoing"
icon: 08-layers
subtitle: "An HPS-style Restricted-Dataflow RISC-V Core"
summary: "Extending the in-order pipeline toward out-of-order execution using HPS's restricted-dataflow model — node tables tracking a dataflow graph, not classic reservation stations."
description: "Designing and implementing an out-of-order RV32IM core based on Patt, Hwu, and Shebanow's HPS microarchitecture (MICRO-18, 1985): instructions become nodes in a restricted dataflow graph tracked via node tables, rather than Tomasulo-style reservation stations, on top of the Foundation Core pipeline."
tags: [Verilog, SystemVerilog, UVM]
tracks: [out-of-order, verification, ppa-analysis]
---
Extending the in-order [Foundation Core](/projects/foundation-core/) pipeline
toward out-of-order execution — not via classic Tomasulo/scoreboarding, but
based on Yale Patt, Wen-mei Hwu, and Michael Shebanow's
["HPS, A New Microarchitecture: Rationale and Introduction"](https://dl.acm.org/doi/10.1109/MICRO.1985.13432)
(MICRO-18, 1985). HPS treats the program as a restricted dataflow graph:
instructions become nodes tracked in node tables, waiting on their operands
to become available, rather than being held in reservation stations against
a fixed instruction-level structure.

See the [Out-of-Order Execution](/tracks/out-of-order/) track for weekly
progress.
