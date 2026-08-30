---
title: "Week 3: Real numbers — Nangate45 synthesis and OpenSTA timing/power"
date: 2026-03-23
project: sysmac
tracks: [ppa-analysis, systolic-array]
summary: "Generic gate counts disagree with real Nangate45 area on which dataflow is smaller — and output-stationary turns out ~38% lower power despite near-identical area."
---
This is the week the project stops being "does it compute the right matmul"
and becomes "which of these two dataflows is actually better in silicon."
Two synthesis passes and one static-timing/power run later, there's a real
answer — and it's more interesting than either dataflow just winning
outright.

## First: fixing RTL the toolchain couldn't handle

Yosys's frontend can't parse the immediate-assertion syntax in `mac_pe.sv`/
`mac_pe_os.sv` at all — and assertions are simulation-only anyway, no
hardware meaning. Both synthesis scripts pass `-D SYNTHESIS` to
`read_verilog`, and the RTL's `` `ifndef SYNTHESIS `` guard strips that block
out before Yosys ever sees it:

```tcl
read_verilog -sv -D SYNTHESIS rtl/mac_pe.sv rtl/mac_array.sv
synth -top mac_array
```

## Generic synthesis: a gate-count PPA proxy, and why not to trust it yet

Ran both variants through Yosys's internal (technology-independent) cell
library first — cheap, fast, no PDK required:

| | Weight-stationary | Output-stationary |
|---|---|---|
| Total cells (16 PEs) | 14,432 | 15,232 (+5.5%) |

Read literally, this says output-stationary costs more silicon. Kept that
number, moved on to real synthesis, and it turned out to matter a lot that
this was flagged as provisional rather than reported as the answer.

## Nangate45-mapped synthesis: the real numbers

Both variants synthesized against the same 45nm open standard-cell library
(via OpenROAD-flow-scripts), same `synth/constraints.sdc` (10ns clock, 2ns
I/O delay) held identical across both runs so the comparison stays
apples-to-apples:

```
$ yosys synth_nangate.ys
...
   Chip area for top module '\mac_array': 19679.744000
     of which used for sequential elements: 4170.880000 (21.19%)

$ yosys synth_nangate_os.ys
...
   Chip area for top module '\mac_array_os': 19496.736000
     of which used for sequential elements: 4085.760000 (20.96%)
```

| | Weight-stationary | Output-stationary |
|---|---|---|
| Area | 19,679.7 µm² | 19,496.7 µm² (**−0.9%**) |

**The generic gate count and the real mapped area disagree on direction.**
Generic synthesis said output-stationary costs 5.5% *more* gates; the real
Nangate45-mapped area says it's 0.9% *smaller*. Not a contradiction — this is
the entire reason real synthesis exists. Gate *count* ignores that different
cell types (an `AOI21_X1` vs. an `XNOR2_X1`) have very different real
silicon area, and ABC's technology mapping picks a different mix of cells
for each netlist depending on what it can pattern-match. A generic gate
count is not a reliable area proxy — that's a finding about methodology,
not just about these two dataflows.

## OpenSTA: timing and power

Read the Nangate45 netlist plus the shared `.sdc`, ran `report_checks`,
`report_worst_slack`, and `report_power` (flat 0.5 switching-activity
assumption via `set_power_activity`) against both:

```
$ sta -exit synth/sta_ws.tcl
...
worst slack max 6.18
worst slack min 0.08
Total    2.27e-02   1.51e-02   4.32e-04   3.82e-02 100.0%

$ sta -exit synth/sta_os.tcl
...
worst slack max 6.05
worst slack min 0.10
Total    1.31e-02   1.02e-02   4.36e-04   2.36e-02 100.0%
```

| | Weight-stationary | Output-stationary |
|---|---|---|
| Worst setup slack (10ns clock) | 6.18 ns | 6.05 ns |
| Worst hold slack | 0.08 ns | 0.10 ns |
| Critical path delay | ~3.79 ns | ~3.91 ns |
| Total power (0.5 activity factor) | 38.2 mW | **23.6 mW (−38%)** |

Timing is comfortably met for both at the 10ns target — weight-stationary's
critical path is marginally shorter, but both have several nanoseconds of
slack to spare, so timing isn't the differentiator here. **Power is.**
Output-stationary comes in ~38% lower power despite near-identical area —
the real, substantive PPA difference between these two dataflows for this
workload, and not something obvious from the RTL alone. (Caveat worth
repeating exactly because it's easy to skim past: this power number uses a
flat activity-factor assumption, not activity derived from an actual
simulation/VCD — a fair comparison between the two designs under identical
methodology, but not a precise absolute number for either.)

## Seeing the placement, not just the numbers

<figure>
  <img src="/assets/images/sysmac/mac-array-placement-openroad-gui.png" alt="OpenROAD GUI showing the placed weight-stationary mac_array die, with each of the 4 systolic PE rows highlighted a different color: pink (row 0) at top, teal (row 1) and yellow (row 2) in the middle two bands, green (row 3) at bottom, in organic, non-rectangular blobs rather than clean stripes">
  <figcaption>Live from <strong>OpenROAD</strong> after floorplanning + global placement (14,176 cells, 100% legal on the first legalization pass), with each of the array's 4 systolic rows selected and highlighted a different color: pink = row 0, teal = row 1, yellow = row 2, green = row 3.</figcaption>
</figure>

Two things worth reading directly off this image rather than skimming past:

**The four colors stack roughly top-to-bottom in row order — not a
coincidence.** `psum_out` chains row `i` to row `i+1` vertically (PE(i,j)'s
partial sum feeds directly into PE(i+1,j) below it), so each row is only
*directly* wired to its immediate neighbors. Global placement minimizes
total wirelength, and minimizing wirelength for a chain naturally pulls
logically-adjacent links physically close together — nothing told the
placer "row 0 goes on top," that ordering fell out of the real connectivity.

**The boundaries are organic blobs, not clean rectangles — also not a
placement quirk.** Within a row, `act_in`→`act_out` chains each PE to its
immediate horizontal neighbor, which is why same-row cells cluster at all.
But `weight_load`, `clk`, and `rst_n` fan out identically to all 16 PEs at
once — one global signal touching every cell in the array, pulling
everything toward a shared centroid and blurring any clean partition line.
What's visible here is that tension resolved by continuous wirelength
minimization (Nesterov global placement), not a hard row constraint — cells
settle wherever total wire length is lowest, and that's an organic boundary,
not a rectangle.

Also worth noting how this differs from the Foundation Core's synthesis
story: that core's placement view was dominated by one memory (96% of the
die, because Yosys never inferred a real SRAM macro for it). This array has
no such outlier — every PE is the same size, so the picture above is purely
about *connectivity*, not one module drowning out the rest.

## Next

- Push the "which dataflow is faster" question past static timing and into
  actual measured throughput — cycles to a full result, not just whether a
  single pass meets the clock.
- Revisit whether weight-stationary's "tolerates unskewed feeding" claim
  from Week 2 actually survives real back-to-back streaming, or only looked
  true because of how it was tested.
