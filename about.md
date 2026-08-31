---
layout: page
title: About
permalink: /about/
---
I'm an Electrical and Computer Engineering student at the University of
Texas at Austin, building RISC-V cores and doing computer architecture
research — from a from-scratch in-order pipeline through an HPS-style
out-of-order extension — alongside FPGA accelerator research in a computer
architecture lab and firmware bring-up on a real UAV.

<p><a class="button" href="{{ '/assets/files/Tanooj_Kanike_Resume.docx' | relative_url }}" download>Download Resume</a></p>

## Background
B.S. Electrical and Computer Engineering Honors at UT Austin (expected May
2028, GPA 3.97). Relevant coursework includes Digital Logic Design (ECE
316) — combinational and sequential logic in Verilog, implemented on an
Artix-7 Basys3 FPGA via Xilinx Vivado — and Computer Architecture under
Prof. Yale Patt, covering pipelining, out-of-order execution, and register
renaming. [NodeFlow](/projects/nodeflow/)'s HPS-based design is a direct
extension of that coursework: Patt is also the microarchitect behind the
HPS papers NodeFlow is built from.

## Current position
Air Systems Engineering Intern at Infravision (Buda, TX), May 2026–present,
on the Valkyrie 100 UAV team. Diagnosed bit-timing faults with an
oscilloscope and built the project's DroneCAN stack in bare-metal C,
producing the firmware's first valid CAN frames and unblocking bring-up
across 5 PCBs (power distribution, payload control, and others). Extended
scope beyond the original assignment into GPS integration, radar testing,
secondary radio bring-up, and gimbal/payload integration — including
self-directed ethernet switch, load cell, and SSI encoder validation, and
resolving I2C address collisions and ADC current-sense calibration issues
while coordinating daily with a six-person cross-functional team.

## Lab / research position
Undergraduate researcher in the Laboratory for Computer Architecture under
Prof. Lizy John, UT Austin, Nov 2025–present. Designing pipelined RTL for
an FPGA accelerator implementing LL-ViT — replacing MLP layers with
LUT-based channel mixers — and assisting graduate researchers with FPGA
synthesis and timing analysis in Xilinx Vivado, demonstrating 1.9x energy
efficiency and 1.3x lower latency over baseline. Also developing RTL for a
Differentiable Weightless Neural Network (DWN) inference accelerator
targeting on-chip bio-impedance signal processing for blood-pressure
estimation on edge ASICs. Presents weekly out-of-order execution research
to a PhD-led computer architecture panel — the same research this site's
[NodeFlow](/projects/nodeflow/) project applies to an independent RISC-V
core.

## Skills
- **RTL & physical design:** SystemVerilog, Verilog, Yosys, OpenROAD,
  Verilator, OpenSTA, Xilinx Vivado, Artix-7, pipelining, fixed-point
  arithmetic, UVM, gem5
- **Embedded & firmware:** bare-metal C, DroneCAN/CAN protocols,
  oscilloscope debugging, I2C, ADC calibration, UART, PX4, ROS2, Linux
- **Software & tools:** C/C++, Python, PyTorch, MATLAB, Git, Docker,
  roofline modeling

## Elsewhere
- [GitHub](https://github.com/{{ site.github_username }})
- [LinkedIn](https://linkedin.com/in/{{ site.linkedin_username }})
- [{{ site.email }}](mailto:{{ site.email }})
