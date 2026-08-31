---
title: "Week 5: Yeh & Patt's two-level predictor, wired in and measured"
date: 2026-04-17
project: foundation-core
tracks: [branch-prediction]
summary: "GAg is live in the pipeline, it had a real bug of its own, and it's now measured — not estimated — on 20 RTL test programs."
---
Last week's post was still deciding what to build. This week is the answer:
a real two-level adaptive predictor, wired into the pipeline, with its own
bug, and measured on real hardware signals instead of a synthetic trace.

## The baseline it's replacing

Before any of this, `pipeline_top.sv` had no predictor structure at all.
`pc_next` defaulted to `pc_plus4` and only took `mem_pc_target` when `pc_src`
fired in the MEM stage — every branch was implicitly predicted not-taken,
and every taken branch cost a fixed flush regardless of history, because
nothing existed that could do better.

## Why the two-level adaptive scheme, and not gshare

The options on the table last week were static not-taken, a 1-bit predictor,
a 2-bit saturating counter, and something folding in global history. That
last option is where the naming gets confusing, so worth being precise:
what got built is **Yeh & Patt's two-level adaptive training scheme**, GAg
variant — not gshare. They're related but distinct. Gshare XORs the branch's
own PC into the global history to index its pattern table, so different
branches with the same history can land in different table entries. GAg
skips the PC entirely: one global Branch History Register indexes one global
Pattern Table, full stop, so *every* branch in the program shares the same
index space. That's a real tradeoff (see the aliasing result below), not an
implementation shortcut — GAg is the base scheme from the original paper,
and gshare is a later refinement of it.

<figure>
  <img src="/assets/images/foundation-core/yeh-patt-two-level-adaptive-structure.png" alt="Diagram of the Two-Level Adaptive Training scheme: a Branch History Register indexes a Pattern Table of state-machine counters, whose state feeds both a prediction and a state-transition update">
  <figcaption>Figure 1 from Yeh, T.-Y. and Patt, Y. N., "Two-Level Adaptive Training Branch Prediction," <em>Proceedings of the 24th Annual International Symposium on Microarchitecture (MICRO-24)</em>, 1991 — the structure <code>bp_two_level.sv</code> is a direct RTL implementation of: history register indexes a pattern table, whose counter state both predicts and re-trains itself.</figcaption>
</figure>

`bp_two_level.sv` maps onto that figure almost register-for-register: the
paper's Branch History Register is `bhr`, the Pattern Table is `pht` (2-bit
saturating counters instead of the paper's general state machine), and the
state-transition logic `δ` is just the increment/decrement on
`update_taken`. Wired into `pipeline_top.sv` with `BHR_WIDTH=12` — a
4096-entry PHT, chosen as an early ID-stage redirect gated by a new
`predictor_enable` port, added only after three unrelated, pre-existing
pipeline bugs surfaced and got fixed first (bad `bne` decoding, `addi`
misdecoding as subtract, and a stale-register-read WB/ID collision) — none
of those three are predictor bugs, just things that had to be true before
trusting any predictor measurement built on top of them.

## The bug: training the wrong table entry

The one real bug in the predictor itself: the PHT update was indexed with
the **live** BHR value instead of the BHR snapshot from when the branch was
actually predicted. Pipeline latency between fetch and resolve means the
live BHR has usually already been shifted by other branches resolving in
between — so the update was training whichever PHT entry the history
happened to point at *by the time the branch resolved*, not the entry that
actually produced that branch's prediction. Fixed by threading a
`predict_bhr` snapshot through the pipeline alongside the prediction itself
and using that snapshot (`update_bhr`) for the read-modify-write, while the
live `bhr` register still shifts in true chronological order regardless of
which entry any individual update touches — the snapshot and the live
shift-register are solving two different problems and both have to be
right independently.

## Verification: 20 programs, not one loop

Trusting a performance number out of this required trusting correctness
first. Every one of 20 hand-written RISC-V programs — loop-heavy,
branch-heavy/correlated, mixed control flow, and deliberately adversarial —
went through the same four-step check: assemble, run through the
`single_cycle.cpp` oracle for ground truth, run through the pipelined RTL
with the predictor on, run again with it forced off, and only trust that
program's cycle counts if both RTL runs matched the oracle exactly.

**20/20 matched on both configurations.** One apparent mismatch, on the
bubble-sort program, turned out to be signed-vs-unsigned decimal printing of
an identical bit pattern, not a functional bug — confirmed by comparing the
raw hex.

## What the measurement actually showed

Numbers below are real hardware counters out of the RTL via Verilator, not
an analytical estimate:

- The long steady-state loop (520 iterations) is the headline win: 32% fewer
  cycles, misprediction rate falls from 99.8% to 2.7% once the table trains
  up.
- Short one-shot loops and searches show **zero** measured benefit — not a
  bug, a structural consequence of GAg with a 4096-entry table: a loop that
  never revisits the same global-history index before it ends has nothing
  to train on.
- Periodic patterns (period-2, period-5) roughly halve their misprediction
  rate; strict alternation stays stuck near 50% even with the predictor
  active, matching the textbook hard case for saturating counters.
- Both adversarial sequences (an LCG and a real Thue-Morse sequence) land
  right at the ~50% coin-flip floor — genuinely hard, not accidentally
  learnable.
- The cleanest and most informative result: a program built specifically so
  an always-taken branch and an always-not-taken branch alias to the same
  global-history index. Cycles, IPC, and misprediction rate came out
  bit-identical on vs. off — the always-taken branch never once gets
  predicted correctly, because the not-taken branch dominates their shared
  PHT entry at a 12:1 ratio every repetition. That's the direct, measured
  cost of GAg having no per-branch isolation, which gshare's PC-mixed index
  would at least partially separate.

<figure>
  <img src="/assets/images/foundation-core/branch-predictor-results-table.png" alt="Table of RTL-measured cycles, IPC, and misprediction rate for all 20 test programs, predictor on vs. off">
  <figcaption>All 20 programs, on vs. off, straight out of the RTL hardware counters — full breakdown and category descriptions in <code>BRANCH_PREDICTOR_PERFORMANCE.md</code>.</figcaption>
</figure>

**Next:** the 20 program sources and raw run logs currently exist only in
the session scratchpad, not checked into the repo — getting those committed
is the immediate cleanup item. After that, the aliasing result above is the
natural argument for trying gshare or a per-branch (PAg) history next, now
that there's a real measured baseline to compare it against instead of a
synthetic trace.
