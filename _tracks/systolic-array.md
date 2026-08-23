---
title: Systolic Array
slug: systolic-array
summary: A systolic-array matrix-multiply unit as a memory-mapped accelerator off the core.
status: Active
color: "#2FA8B8"
---
Matrix multiplication dominates both classic DSP and ML inference workloads.
This track builds a systolic array (REPLACE_ME: dimensions, e.g. a weight-
stationary N×N PE grid) as a memory-mapped accelerator the core can offload
matmuls to, and measures the speedup versus running the same workload on the
core alone.

**Why:** REPLACE_ME (e.g. "systolic arrays are the backbone of accelerators
like the TPU — wanted to understand the PE design and dataflow tradeoffs by
building one and wiring it into a real processor, not just simulating it in
isolation").
