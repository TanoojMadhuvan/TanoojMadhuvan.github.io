---
title: "Week 4: Skew vs. unskew — correcting a claim and measuring real throughput"
date: 2026-03-30
project: sysmac
tracks: [ppa-analysis, verification, systolic-array]
summary: "Weight-stationary's 'tolerates unskewed feeding' claim only held because every test so far held a value constant. Real back-to-back streaming needed the same row-skewed timing output-stationary required from day one — and the corrected result is a genuine 10-vs-14-cycle throughput measurement."
---
Closing out this arc with a correction, not just a new result. Before getting
to the correction, this post walks through three things step by step, since
they're easy to blur together: what matmul the array is actually doing, what
a "row" physically is, and what "skew" means in concrete cycle-by-cycle
terms — then the actual bug, the proof, and the final measurement.

## Step 1: what A × W actually is here

- This array computes **C = A × W**.
- **W** is the 4×4 weight matrix — loaded once, one element per PE, held
  stationary for the whole computation (Week 1/2).
- **A** is the 4×4 activation matrix — four separate input vectors, one per
  row of A.
- The exact numbers this post's test uses:

  ```
  W (weights, stationary)      A (activations, streamed)
  1 2 0 0                      1 3 5 7
  0 1 2 0                      2 4 6 9
  0 0 1 2                      8 1 3 5
  2 0 0 1                      9 7 2 4

  C = A × W (expected result)
  15  5 11 17
  20  8 14 21
  18 17  5 11
  17 25 16  8
  ```

## Step 2: what "row i" physically means

- The array is a 4×4 grid of PEs. PE(i, j) permanently holds weight
  element `W[i][j]` (Week 2's `generate` loop wires this up directly).
- Feed one activation **vector** into the array by presenting its element
  `k` to **row k** — component 0 of the vector goes into row 0, component 1
  into row 1, and so on.
- Each column `j` of PEs accumulates that vector's dot product against
  column `j` of W as it flows down — column `j`'s output is one element of
  the result vector.
- To multiply the *whole* matrix A (all four rows) through the array, you
  feed its four rows through one after another. **How** "one after another"
  actually works is the entire subject of this post.

## Step 3: two different ways to feed four vectors through

- **Settle-and-read** — what Weeks 1-3 actually tested:
  - Feed one full vector into all 4 rows on the *same* cycle.
  - Hold it steady and wait for the pipeline to fully drain (~7 cycles for
    this size array).
  - Read the finished result off `psum_out`, *then* start the next vector
    from scratch.
  - Correct, but wasteful — the array sits mostly idle, waiting to settle,
    instead of doing new work every cycle.
- **Continuous streaming** — this week's actual goal:
  - Feed a brand-new vector into row 0 every single cycle, with no waiting.
  - Once the pipeline fills up, get one finished output vector out **every
    cycle** instead of one every ~7 cycles.
  - Dramatically faster — *if* fed correctly. Feeding it naively (every row
    getting a new vector on the same cycle, same as settle-and-read but
    without the wait) is exactly the bug this post is about.

## Step 4: what "skew" actually means

- Physical fact: there's a register between every pair of vertically
  stacked PEs. A partial sum takes exactly **1 clock cycle** to travel from
  row `i` to row `i+1` (`mac_pe.sv`'s `psum_out <= psum_in + ...`, latched
  every cycle).
- Consequence: if row 0 injects vector `v`'s contribution into a column's
  running sum at cycle `v`, that partial sum doesn't physically *arrive* at
  row 1 until cycle `v+1`.
- For row 1 to add its **own** correct contribution to vector `v` — not some
  other vector — row 1 has to present vector `v`'s row-1 element at that
  exact cycle, `v+1`: one cycle later than row 0 presented the same vector.
- Generalizing: **row `i` must present vector `v` exactly `i` cycles after
  row 0 does.** That delay, growing by one cycle per row down the array, is
  what "skew" means. In code (`mac_array_tb.cpp`, test 4):

  ```cpp
  // vector index row i is presenting at global cycle s
  int v = s - i;
  act[i] = (v >= 0 && v < N) ? A[v][i] : 0;
  ```

- Concretely, for this post's `A`, here's which row of `A` each array row is
  presenting at each global cycle (`–` = nothing yet, or already done):

  | Cycle | Row 0 | Row 1 | Row 2 | Row 3 |
  |---|---|---|---|---|
  | 0 | A[0] | – | – | – |
  | 1 | A[1] | A[0] | – | – |
  | 2 | A[2] | A[1] | A[0] | – |
  | 3 | A[3] | A[2] | A[1] | A[0] |
  | 4 | – | A[3] | A[2] | A[1] |
  | 5 | – | – | A[3] | A[2] |
  | 6 | – | – | – | A[3] |

  Reading down any diagonal, the same vector (e.g. `A[0]`) shows up in row 0
  at cycle 0, row 1 at cycle 1, row 2 at cycle 2, row 3 at cycle 3 — walking
  diagonally through the grid, one cycle per row, instead of straight down a
  single column. That diagonal walk *is* the skew.

<figure>
  <img src="/assets/images/sysmac/skew-vs-unskew-timing.svg" alt="Diagram comparing unskewed settle-and-read timing (top: all 4 rows hold the same vector constant simultaneously for ~7 cycles) against skewed continuous streaming (bottom: each row's 4 vectors staggered by one cycle per row, output columns landing on consecutive cycles)">
  <figcaption>Unskewed settle-and-read (what Weeks 1-3 actually tested) vs. skewed continuous streaming (what real back-to-back throughput requires) — same array RTL, different driving sequence.</figcaption>
</figure>

## Step 5: why *not* skewing would actually break it

- Without the delay, every row would present vector `v`'s element at the
  same cycle `s = v` — identical to settle-and-read, just without the wait.
- Follow what happens at PE(1, j): row 0 pushed vector `v-1`'s contribution
  into the pipe one cycle earlier (at cycle `v-1`), and it takes 1 cycle to
  arrive — so at cycle `v`, the partial sum sitting at row 1 belongs to
  vector `v-1`, not vector `v`.
- Meanwhile row 1 (unskewed) is presenting vector `v`'s own element at that
  same cycle. PE(1, j) would add **vector v's** product onto **vector
  v-1's** partial sum — two different vectors' results silently merged.
- Only the diagonal case (`i == j`) happens to line up by coincidence —
  everywhere else, the array would multiply mismatched values together and
  get a wrong answer while looking like it's "streaming."

## The claim that needed correcting

`mac_array.sv`'s header comment (Week 2) draws a real contrast: weight-
stationary settles on a *constant* activation vector, while output-
stationary streams a *sequence* through every cycle — and only the sequence
case needs row-skewed timing to stay correct. That's true as far as it goes,
but:

- Every weight-stationary test in Weeks 1-3 used exactly the settle-and-read
  style from Step 3 — one vector, held *constant*, until it settled.
- A stale copy of a constant looks identical to a fresh one on every cycle —
  so those tests could never have exposed a skew bug even if one existed.
  They were testing settle-and-read, never genuine back-to-back streaming.
- The array RTL itself needed **zero changes** for this fix — it already
  behaves correctly either way. Only the *test driver's* input sequence
  needed to follow the skew rule from Step 4.

## Proof, not just derivation

Ran the skewed sequence against Verilator for real, logged every cycle's
output, and searched for each expected result value per output column:

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
Total cycles to a complete 4x4 @ 4x4 result, measured (not estimated)
against Verilator:

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
