---
title: Systolic Array
slug: systolic-array
summary: A weight-stationary vs. output-stationary systolic MAC array, compared on real synthesized PPA numbers.
status: Active
color: "#2FA8B8"
---
Matrix multiplication dominates both classic DSP and ML inference workloads.
This track builds a small systolic MAC array — a 4x4 weight-stationary PE
grid (TPUv1-style) and a second output-stationary variant sized for a fair
comparison — and pushes both through a real Yosys/OpenROAD/OpenSTA flow to
get actual area, power, and measured cycle counts, not just simulated
correctness. See [SysMAC](/projects/sysmac/) for the full four-week build.

**Why:** dataflow choice — which operand stays stationary in the array — is
a real architectural tradeoff that's easy to state in the abstract and hard
to actually measure. Wanted to build both variants and run both through the
same synthesis/STA flow on the same workload, rather than simulate one and
assume how the other would compare.
