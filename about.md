---
layout: page
title: About
permalink: /about/
---
I'm an ECE Honors student at UT Austin, building toward a career in
computer architecture and PPA engineering — the intersection of hardware
design, verification, and physical design that actually decides how fast,
efficient, and power-conscious real silicon ends up being. I like it
because none of those three things is optional: a clever architecture that
doesn't verify is a bug, and one that verifies but never gets synthesized
is a slide deck.

<p><a class="button" href="{{ '/assets/files/Tanooj_Kanike_Resume.docx' | relative_url }}" download>Download Resume</a></p>

<figure>
  <img src="{{ '/assets/images/about-infravision-drone.jpg' | relative_url }}" alt="Tanooj standing behind a large octocopter UAV in a warehouse at Infravision">
  <figcaption>The Valkyrie 100 — the octocopter I do firmware bring-up on at Infravision.</figcaption>
</figure>

## What I've built
I designed and verified a RISC-V processor from scratch — a custom
assembler, a single-cycle core, and a pipelined RTL implementation in
SystemVerilog with hazard detection and forwarding — then pushed it
through Yosys and OpenROAD to see what it actually costs in real silicon,
not just whether it simulates correctly. I'm extending that same core into
[NodeFlow](/projects/nodeflow/), an out-of-order design based on Yale
Patt's HPS microarchitecture instead of the usual Tomasulo playbook, and I
built [SysMAC](/projects/sysmac/) alongside it to study systolic dataflow
tradeoffs (weight-stationary vs. output-stationary) grounded in the TPUv1
paper. [KVRoof](/projects/kv-cache-transformers/) looks at the same
PPA-minded questions from the memory side — modeling when a transformer's
KV-cache stops fitting on-chip.

A bit of the paper trail behind all of it: B.S. Electrical and Computer
Engineering Honors at UT Austin, expected May 2028 (3.97 GPA). The
out-of-order work above traces straight back to Prof. Yale Patt's own
Computer Architecture course — yes, that Yale Patt, the same one behind
the HPS papers NodeFlow is built from.

## In the lab
<img class="lab-logo" src="{{ '/assets/images/lab-computer-architecture-logo.png' | relative_url }}" alt="Laboratory for Computer Architecture logo">

I'm part of Prof. Lizy John's Laboratory for Computer Architecture at UT
Austin, where I work on Differentiable Weightless Neural Networks
(DWNs) — extremely compact architectures designed to fit on edge/ASIC
hardware — applied here to on-chip bio-impedance signal processing for
blood pressure estimation. I've also designed pipelined RTL for an FPGA
accelerator implementing LL-ViT, which showed a 1.9x energy-efficiency and
1.3x latency win over baseline, and I present weekly out-of-order
execution research to a PhD-led panel there — the same research that feeds
directly into NodeFlow.

<figure>
  <img src="{{ '/assets/images/about-lab-dinner.jpg' | relative_url }}" alt="Prof. Lizy John's Laboratory for Computer Architecture group at a dinner">
  <figcaption>Lab dinner, April 2026.</figcaption>
</figure>

## The day job (Infravision)
Outside of school I'm an Air Systems Engineering Intern at Infravision, on
the Valkyrie 100 UAV team — that's the drone above. I've diagnosed
bit-timing bugs from first principles with an oscilloscope, built the
project's DroneCAN stack from bare-metal C, and put together
hardware-verified compliance documentation for DGCA type certification —
work that unblocked board bring-up across all five PCBs on the platform.
Along the way I picked up GPS integration, radar testing, secondary radio
bring-up, and gimbal/payload validation outside my original scope, mostly
by asking where else I could be useful.

## Before all this
FTC robotics (competition programming, computer vision pipelines),
full-stack development, and hackathon leadership, including a top finish
at a Microsoft hackathon, plus a run of mentorships with Microsoft, Cisco,
and Toyota that gave me an early look at cloud infrastructure and product
strategy. None of it was computer architecture, but it's most of why I
like building things enough to be doing this now.

Day to day the toolbox is SystemVerilog, Verilog, Yosys, OpenROAD,
Verilator, OpenSTA, and Vivado on the hardware side; bare-metal C,
DroneCAN/CAN, and PX4/ROS2 on the firmware side; C/C++, Python, and
PyTorch for everything else.

## Elsewhere
- [GitHub](https://github.com/{{ site.github_username }})
- [LinkedIn](https://linkedin.com/in/{{ site.linkedin_username }})
- [{{ site.email }}](mailto:{{ site.email }})
