---
title: "SysMAC"
slug: sysmac
index: "01"
date_range: "May 15–Jun 5, 2026"
icon: 04-memory-grid
subtitle: "Systolic MAC Array"
summary: "A weight-stationary vs. output-stationary PPA comparison on a small systolic MAC array, synthesized through Yosys/Nangate45 and measured with OpenSTA."
description: "A small weight-stationary and output-stationary systolic MAC array in SystemVerilog exploring the TPUv1 dataflow tradeoff at toy scale — real area, timing, and power from a Yosys/Nangate45/OpenSTA flow, plus a measured cycle-time comparison between the two dataflows under continuous streaming."
tags: [SystemVerilog, "C++", Verilator, Yosys, OpenSTA]
tracks: [ppa-analysis, verification, systolic-array]
---
A small weight-stationary and output-stationary systolic MAC array, exploring
the same architectural principle TPUv1 used at datacenter scale — a systolic
dataflow plus INT8 quantization — at a toy 4x4 scale, synthesized and
measured with the same category of tools real physical-design engineers use.

The actual finding is a **PPA comparison between two dataflow variants** of
the same array, held to the same PE count and clock target: output-stationary
comes in ~38% lower power despite near-identical area, while weight-stationary
finishes a full 4x4 @ 4x4 matmul faster under continuous streaming (10 cycles
vs. 14) — a genuine multi-axis tradeoff, not a "one dataflow wins" story.
