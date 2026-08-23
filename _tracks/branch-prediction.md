---
title: Branch Prediction
slug: branch-prediction
summary: Reducing control-hazard penalty in the pipelined core with dynamic branch prediction.
status: Active
color: "#4C8FCB"
---
The pipelined core currently resolves branches in [REPLACE_ME: stage, e.g.
"EX"], costing [REPLACE_ME: N] stall/flush cycles per taken branch. This track
adds dynamic branch prediction — REPLACE_ME: which scheme(s) (e.g. static
predict-not-taken → 2-bit saturating counter → gshare/BTB) — to cut that
penalty and measure the impact on IPC.

**Why:** REPLACE_ME (e.g. "branch prediction is one of the highest-leverage
microarchitecture techniques and a common interview/research topic — wanted
to implement and measure it myself rather than just read about it").
