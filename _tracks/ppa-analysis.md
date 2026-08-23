---
title: PPA Analysis
slug: ppa-analysis
summary: Power, performance, and area characterization of the core through synthesis.
status: Active
color: "#C08A2E"
---
The core already synthesizes through Yosys against the Nangate45 standard-cell
library (see [/foundation](/foundation/)). This track digs into the resulting
power/performance/area (PPA) numbers — critical path, cell area breakdown by
module, and how architectural choices (e.g. pipelining, forwarding logic)
trade off against them.

**Why:** REPLACE_ME (e.g. "RTL that 'works' in simulation isn't the same as
RTL that's efficient in silicon — wanted hands-on experience with the
synthesis-to-PPA feedback loop that real hardware teams run on").
