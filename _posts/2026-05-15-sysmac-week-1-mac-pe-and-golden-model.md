---
title: "Week 1: A weight-stationary MAC PE, an overflow assertion, and a golden model"
date: 2026-05-15
project: sysmac
tracks: [verification, systolic-array]
summary: "Starting a systolic-array side project after a short break — the atomic PE tile, an immediate-assertion overflow check, and why the single-PE golden model barely earns its keep."
---
Picking up a side project after two weeks off from the [Foundation Core](/projects/foundation-core/)
arc: **SysMAC**, a small systolic MAC array motivated by the TPUv1 paper
(Jouppi et al., ["In-Datacenter Performance Analysis of a Tensor Processing
Unit"](https://arxiv.org/abs/1704.04760)). Being upfront about scope from day
one: this is a 4x4 array run through an educational Yosys/OpenROAD flow, not
a 256x256 datacenter accelerator on a production node. The point isn't to
reproduce Google's numbers — it's to measure the *direction* of an
architectural tradeoff (dataflow choice) myself, with the same category of
tools real PD engineers use.

## Why a systolic array, and why weight-stationary first

A systolic array trades data movement for local reuse: instead of reading
every operand from memory every cycle, a value loads once and gets reused by
a chain of neighboring processing elements as data flows past it. There are a
few ways to decide *which* operand stays put — weight-stationary holds the
weight, output-stationary holds the accumulator, input-stationary holds the
activation. Starting with weight-stationary because it's the one TPUv1 itself
uses, and because — on paper, before actually measuring it — it seemed like
the simplest one to reason about: the weight loads once and never moves
again for the whole matmul.

## The atomic tile: `mac_pe.sv`

<figure>
  <img src="/assets/images/sysmac/mac-pe-weight-stationary.png" alt="Diagram of a weight-stationary PE(i,j): a weight register loaded once with w = B[i][j] and held stationary, feeding a datapath that every cycle computes psum_out = psum_in + a_in × w, with a_in arriving from the left PE and passed through to a_out, and psum_in/psum_out flowing vertically to the PE above/below">
  <figcaption>Weight-stationary PE: the weight register loads once and never changes; activation and partial sum are what move, left-to-right and top-to-bottom respectively.</figcaption>
</figure>

Single INT8 x INT8 -> 32-bit-accumulate processing element. Weight loads once
via `weight_load` and sits in a register; activations stream left-to-right
(`act_in` -> `act_out`); partial sums stream top-to-bottom (`psum_in` ->
`psum_out`):

<figure>
  <img src="/assets/images/sysmac/mac-pe-register-block.png" alt="Code screenshot of mac_pe.sv's main always_ff block: on reset, act_out, psum_out, and valid_out clear to zero; otherwise act_out latches act_in, psum_out latches psum_in plus act_in times weight_reg, and valid_out latches valid_in">
  <figcaption><code>mac_pe.sv</code>'s main clocked block: activation and valid pass straight through a register each cycle, while psum accumulates the local multiply on the way through.</figcaption>
</figure>

`valid` travels alongside `act`/`psum` so downstream PEs know when the data
passing through is real versus a reset-time bubble — this becomes load-bearing
once the array is deep enough that not every cycle carries live data (more on
that in Week 4).

## An overflow assertion that isn't proof the math is right

Added an immediate assertion inside the same always block, checking the
accumulate against a 64-bit `longint` computed *before* truncation into the
32-bit register — checking the already-wrapped value would be checking the
exact number that already lost the information needed to catch overflow:

<figure>
  <img src="/assets/images/sysmac/mac-pe-overflow-assertion.png" alt="Code screenshot of mac_pe.sv's overflow assertion: guarded by ifndef SYNTHESIS, it recomputes the accumulate as a 64-bit signed longint ext_sum before truncation and asserts it fits the signed ACC_W-bit range, else errors with mac_pe: accumulator overflow at time">
  <figcaption>The overflow check, stripped out before synthesis ever sees it: the sum is recomputed in 64 bits so overflow can be caught before the real 32-bit add truncates it away.</figcaption>
</figure>

Worth being precise about what this does and doesn't prove: it's a
**protocol/correctness invariant**, not a functional check. It catches "the
design silently wrapped around instead of producing a real number" — it says
nothing about whether the *value* is the matmul result it should be. That's
what the golden model is for.

## First toolchain wall: no concurrent SVA

Reached for `property` / `assert property` first, the standard way to write
this kind of temporal check. Hard syntax error — the available Verilator
(4.038, 2020) doesn't parse concurrent SVA at all. Rewrote as the immediate
assertion above instead (`assert (...) else $error(...)` inside `always_ff`).
Functionally equivalent here since the check is same-cycle, not a
multi-cycle sequence — but worth remembering this isn't true in general;
concurrent properties exist specifically for sequences immediate assertions
can't express.

Second wall right behind it: a comment containing the word "verilator"
(any case, anywhere) gets parsed as a lint pragma and errors if it doesn't
match a real one. Cost a few minutes of confused searching before realizing
the RTL comments themselves were the problem — now avoided deliberately in
every comment in this repo.

## The golden model: honest about which one is doing real work

`golden/mac_pe_golden.py` exists mostly as a documented placeholder. For a
single PE, `psum_out = psum_in + act_in * weight` is hand-checkable by eye —
this script isn't catching bugs a human wouldn't. It earns its keep once
there's an actual array to check (`golden/mac_array_golden.py`, real oracle
via `numpy`, arriving next week) — flagging that distinction now rather than
implying more verification rigor than a one-line golden model for a
one-line datapath actually provides.

## Running it for real

```
=== mac_pe Testbench ===
PASS weight=5 act=3 psum_in=0 -> psum_out=15
PASS chained psum_in=15 act=4 weight=5 -> psum_out=35
PASS valid_in=0 -> valid_out=0 (bubble)
PASS weight reload to -2, act=10 -> psum_out=-20
=== 4 passed, 0 failed ===
```

Four directed tests, hand-derived against the golden model: a basic
weight/act multiply, a chained `psum_in` (as if fed from the PE above), a
`valid_in=0` bubble propagating correctly, and a weight reload exercising the
signed path (negative weight, since INT8 activations/weights are signed).

Also hit two toolchain gotchas that don't affect correctness but did cost
build-debugging time, worth logging before they need rediscovering:

- Using `$time` in an RTL `$error()` requires the C++ testbench to define
  `double sc_time_stamp()` — nothing to do with SystemC despite the name —
  or the link fails with `undefined reference to sc_time_stamp()`.
- Multi-file Verilator builds default to the *first-listed* file's module as
  top, not whichever module isn't instantiated by anything else. Not an
  issue yet with a single-file `mac_pe` build, but noting it now because it
  will matter the moment `mac_array.sv` instantiates `mac_pe.sv` next week.

**Next:** replicate `mac_pe` into an NxN array via `generate`, build the
array-level golden model that actually does real verification work, and get
a full matmul proven — not just a single vector.
