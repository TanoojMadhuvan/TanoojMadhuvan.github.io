---
title: "Week 3: Splitting the datapath into a 5-stage pipeline"
date: 2026-04-03
project: foundation-core
summary: "IF/ID/EX/MEM/WB, four pipeline registers, and figuring out exactly what a taken branch needs to squash."
---
Took the single-cycle datapath from last week and cut it into the standard
five stages — IF, ID, EX, MEM, WB — connected by four pipeline registers:
`pipe_ifid`, `pipe_idex`, `pipe_exmem`, `pipe_memwb`. `pipeline_top.sv` wires
the same functional units from the single-cycle core (`alu`, `alu_control`,
`control`, `regfile`, `immgen`, `imem`, `dmem`) into this new staged shape —
reusing rather than rewriting felt like the right call, since the ALU and
control logic don't care how many pipeline registers are between them and the
next instruction.

## Where branches resolve

The main design decision this week: branches resolve in the MEM stage, not
EX. `pc_src` is `mem_branch & mem_alu_zero` — both signals already latched
into the EX/MEM register by the time they're used. That means when a branch
is taken, three younger instructions are already in flight and wrong:
whatever's in IF/ID, ID/EX, and EX/MEM. All three get flushed off the same
`pc_src` signal — IF/ID via `if_flush`, ID/EX via `id_flush | pc_src` (ORed
with whatever the hazard unit already wants), and EX/MEM directly via
`pc_src`. Getting that fan-out right took some staring at a piece of paper —
there's actually a comment sitting in `pipeline_top.sv` right above the
branch-decision block that's basically me thinking out loud mid-implementation:

```systemverilog
// also need to flush ID/EX  (instruction decoded after branch)
// id_flush is already driven by hazard unit — we need to OR in branch flush
```

Left it in instead of cleaning it up — it's a more honest record of the
reasoning than a tidied-up version would be.

## What's not tested yet

`pipeline_tb.cpp` runs reset, then 10,000 clock cycles unconditionally, then
dumps whatever's nonzero in the register file and the first 16 words of data
memory. It doesn't check that output against anything — I'm reading it and
deciding for myself whether it looks right. That's fine for now with an ISA
this small, but it's not going to scale past a handful of test programs.

The first version of this only flushed **IF/ID and EX/MEM** off `pc_src` —
squash whatever's newest, and squash whatever's about to write back, which
felt like "both ends" of the pipeline covered. What I missed: the
instruction sitting in **ID/EX** — two instructions past the branch, one
stage further along than felt intuitive to double-check — was also wrong,
and nothing was flushing it. Symptom: a taken branch followed by `add`,
`or`, `sub` left `or`'s destination register holding a value it should
never have computed, even though `if_flush` and `pc_src` were both firing
exactly when expected — `add` (in EX/MEM) was already being caught. Found it
by listing, cycle by cycle, which stage held which instruction the moment
`pc_src` fired — IF/ID, ID/EX, and EX/MEM were all still occupied by
younger, wrong instructions, not just the two I was already flushing. Fix:
OR `pc_src` into `id_flush` too (the comment quoted above is what's left of
that realization).

<figure>
  <img src="/assets/images/foundation-core/id-ex-flush-bug.png" alt="Pipeline diagram showing the or instruction incorrectly continuing to execute after a taken branch, because only IF/ID and EX/MEM were flushed">
  <figcaption>Before the fix: <code>beq</code> resolves taken in MEM at cycle 4, correctly squashing <code>sub</code> (IF/ID) and <code>add</code> (EX/MEM) — but <code>or</code>, sitting in ID/EX, was missed and keeps running (red) all the way to WB.</figcaption>
</figure>

<figure>
  <img src="/assets/images/foundation-core/id-ex-flush-fixed.png" alt="Pipeline diagram showing the or instruction correctly turned into a bubble after ORing pc_src into id_flush">
  <figcaption>After ORing <code>pc_src</code> into <code>id_flush</code>: <code>or</code> becomes a bubble the same cycle the branch resolves, alongside the already-correct <code>sub</code> and <code>add</code> squashes.</figcaption>
</figure>

**Next:** the flush chain is untested beyond "the register dump ends up
looking sane" — want at least a few directed test programs that specifically
put a branch back-to-back with a load or another branch, before trusting this
more.

The `contmul`/`done` loop in `assembly.txt` already has exactly the branch
this needs: `beq x0, x0, contmul` is unconditionally taken every iteration,
so `pc_src` fires like clockwork. Traced it off the actual RTL:

```
cyc  1  pc=0x04  mem_branch=0  pc_src=0  if_flush=0  id_flush=0
cyc  2  pc=0x08  mem_branch=0  pc_src=0  if_flush=0  id_flush=0
cyc  3  pc=0x0c  mem_branch=0  pc_src=0  if_flush=0  id_flush=0
cyc  4  pc=0x10  mem_branch=0  pc_src=0  if_flush=0  id_flush=0
cyc  5  pc=0x14  mem_branch=0  pc_src=0  if_flush=0  id_flush=0
cyc  6  pc=0x18  mem_branch=0  pc_src=0  if_flush=0  id_flush=0
cyc  7  pc=0x1c  mem_branch=1  pc_src=0  if_flush=0  id_flush=0
cyc  8  pc=0x20  mem_branch=0  pc_src=0  if_flush=0  id_flush=0
cyc  9  pc=0x24  mem_branch=0  pc_src=0  if_flush=0  id_flush=0
cyc 10  pc=0x28  mem_branch=1  pc_src=1  if_flush=1  id_flush=1
cyc 11  pc=0x10  mem_branch=0  pc_src=0  if_flush=0  id_flush=0
cyc 12  pc=0x14  mem_branch=0  pc_src=0  if_flush=0  id_flush=0
cyc 13  pc=0x18  mem_branch=0  pc_src=0  if_flush=0  id_flush=0
cyc 14  pc=0x1c  mem_branch=1  pc_src=0  if_flush=0  id_flush=0
cyc 15  pc=0x20  mem_branch=0  pc_src=0  if_flush=0  id_flush=0
cyc 16  pc=0x24  mem_branch=0  pc_src=0  if_flush=0  id_flush=0
cyc 17  pc=0x28  mem_branch=1  pc_src=1  if_flush=1  id_flush=1
```

(Run with `predictor_enable=0` — this predictor doesn't exist yet at this
point in the project, so this reproduces exactly the baseline `pc_src =
mem_branch & mem_alu_zero` behavior described above.) `mem_branch` alone
pulses on *every* branch reaching MEM (cycles 7, 10, 14, 17 — both the
conditional `beq x2,x0,done` and the unconditional `beq x0,x0,contmul`), but
`pc_src`/`if_flush`/`id_flush` only pulse on cycles 10 and 17 — the two
cycles the *unconditional* branch resolves and the misprediction is real.
`pc` visibly jumps back from `0x28` to `0x10` the cycle right after.

Worth being explicit about *why* `pc_src`, `if_flush`, and `id_flush` all
move on the exact same edge instead of rippling in one at a time: only
`mem_branch`/`mem_alu_zero` are actually registered (latched into EX/MEM on
`posedge clk`). `pc_src`, `if_flush`, and `id_flush` are each just
combinational logic sitting directly on top of that one register — no flip-flop
between any of them — so the instant the branch's outcome lands in EX/MEM,
all three settle to their new values within that same cycle. It's one
registered event with an instantaneous combinational fan-out, not four
separately-timed things that happen to line up.

<figure>
  <img src="/assets/images/foundation-core/branch-flush-waveform.svg" alt="Waveform diagram of if_flush, id_flush, and pc_src firing together when a taken branch resolves in MEM">
  <figcaption>Same trace as a waveform — <code>pc_src</code>, <code>if_flush</code>, and <code>id_flush</code> all rising on the same <code>posedge clk</code> in cycles 10 and 17, right as the unconditional backward branch resolves in MEM and the PC bus jumps back to <code>0x10</code>. Values change right at each rising edge, held flat until the next one.</figcaption>
</figure>
