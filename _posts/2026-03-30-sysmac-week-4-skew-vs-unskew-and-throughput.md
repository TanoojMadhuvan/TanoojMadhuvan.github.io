---
title: "Week 4: Skew vs. unskew — correcting a claim and measuring real throughput"
date: 2026-03-30
project: sysmac
tracks: [ppa-analysis, verification, systolic-array]
summary: "Weight-stationary's 'tolerates unskewed feeding' claim only held because every test so far held a value constant. Real back-to-back streaming needed the same row-skewed timing output-stationary required from day one — and the corrected result is a genuine 10-vs-14-cycle throughput measurement."
---
Closing out this arc with a correction, not just a new result. Week 2 said
weight-stationary "tolerates unskewed feeding" — true, but only under a
condition narrow enough that it was hiding something. Fixing that, and
turning it into the actual centerpiece measurement: real, measured cycle
counts for both dataflows on the identical workload.

## The claim that needed correcting

`mac_array.sv`'s header comment (Week 2) draws a real contrast: weight-
stationary settles on a *constant* activation vector, while output-
stationary streams a *sequence* through every cycle — and only the sequence
case needs row-skewed timing to stay correct. That's true as far as it goes.
The gap: every weight-stationary test so far (Week 2's tests 1-3) only ever
held one vector *constant* and read the settled result. A stale copy of a
constant is indistinguishable from the current value — so those tests could
never have exposed a timing bug even if one existed. They weren't testing
streaming at all; they were testing settle-and-read.

Genuinely streaming *different* vectors back-to-back, one per cycle, exposes
the exact same row-to-row misalignment output-stationary has. The fix is the
same derivation output-stationary needed from Week 2: row `i`'s input must
be delayed by `i` cycles relative to row 0's.

```cpp
// vector index row i is presenting at global cycle s
int v = s - i;
act[i] = (v >= 0 && v < N) ? A[v][i] : 0;
```

Row 0 presents vectors 0,1,2,3 at cycles 0-3. Row 1 presents the same four
vectors one cycle later, at cycles 1-4. Row 3 doesn't start until cycle 3.
Nothing about `mac_array.sv` itself changes for this — the array RTL is
identical to Week 2's. Only the *driving sequence* does, which is exactly
why the bug was invisible until a test finally fed genuinely different data
in genuinely back-to-back fashion.

<figure>
  <img src="/assets/images/sysmac/skew-vs-unskew-timing.svg" alt="Diagram comparing unskewed settle-and-read timing (top: all 4 rows hold the same vector constant simultaneously for ~7 cycles) against skewed continuous streaming (bottom: each row's 4 vectors staggered by one cycle per row, output columns landing on consecutive cycles)">
  <figcaption>Unskewed settle-and-read (what Weeks 1-3 actually tested) vs. skewed continuous streaming (what real back-to-back throughput requires) — same array RTL, different driving sequence.</figcaption>
</figure>

## Proof, not just derivation

Ran it against Verilator for real and logged every cycle's output, then
searched for each expected value per output column:

```
column 0 matches at cycles: 3 4 5 6
column 1 matches at cycles: 4 5 6 7
column 2 matches at cycles: 5 6 7 8
column 3 matches at cycles: 6 7 8 9
PASS continuous streaming: all 4 vectors x 4 columns land on consecutive cycles (true 1 result/cycle throughput)
```

Every column's four results land on four *consecutive* cycles — not just
eventually correct with gaps somewhere in between. That distinction is the
actual point: a design that gets the right answer several cycles late,
scattered across a longer window, has not demonstrated 1-result-per-cycle
throughput, even if every individual value is right.

## The measurement this was all building toward

Both dataflows now proven correct on genuinely streamed, back-to-back input.
Total cycles to a complete 4x4 @ 4x4 result, measured (not estimated) against
Verilator:

| | Weight-stationary (streaming) | Output-stationary |
|---|---|---|
| Cycles to full result | ~10 | 14 |

Weight-stationary's pipelined streaming finishes this workload faster than
output-stationary's single-pass reduction — even though [output-stationary
won decisively on power](/blog/2026/03/23/sysmac-week-3-synthesis-and-real-ppa-numbers/)
(~38% lower, near-identical area). Putting both weeks' results next to each
other is the actual conclusion of this project:

| | Weight-stationary | Output-stationary |
|---|---|---|
| Area | 19,679.7 µm² | 19,496.7 µm² (−0.9%) |
| Power (0.5 activity) | 38.2 mW | 23.6 mW (−38%) |
| Cycles to full result | ~10 | 14 |

Neither dataflow wins outright. That's the finding — not "output-stationary
is better" or "weight-stationary is better," but a genuine multi-axis PPA
trade for this specific workload shape: lower power vs. fewer cycles. (The
array RTL is unchanged for weight-stationary's streaming case, so this
doesn't touch the area/power numbers from Week 3 — those still hold exactly.)

## What this project actually proved

Going back to the question from [Week 1](/blog/2026/03/09/sysmac-week-1-mac-pe-and-golden-model/):
is a small, honestly-scoped systolic array project worth doing when it can't
match TPUv1's real numbers? The answer turned out to be yes, for a reason
that has nothing to do with matching those numbers — every real finding here
came from actually running the tools and actually measuring, not from
assuming what a "systolic array project" is supposed to show:

- Generic gate count and real mapped area *disagreed on direction* for the
  same comparison — caught by actually running both, not by trusting either
  one.
- A claim that looked true (Week 2) turned out to be true only under a
  narrower condition than stated — caught by pushing the test past
  settle-and-read into genuine back-to-back streaming.
- The two dataflows don't have a single winner — power and throughput pull
  in opposite directions, which only a real multi-axis measurement shows.

**Stretch, not pursued here:** runtime-configurable precision, functional
coverage. Both real follow-ons, neither blocking the comparison this project
set out to make.
