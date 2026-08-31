---
title: "Week 4: Forwarding and hazard detection"
date: 2026-04-10
project: foundation-core
summary: "Closing the data-hazard gaps forwarding structurally can't cover, and confirming the store-data path was already right."
---
Two real pieces of RTL this week — `forwarding_unit.sv` and `hazard_unit.sv`.

## Forwarding: two sources, priority to the more recent one

`forwarding_unit.sv` compares `ex_rs1`/`ex_rs2` against both `mem_rd` (the
instruction one stage ahead, in EX/MEM) and `wb_rd` (two stages ahead, in
MEM/WB), and drives a 2-bit select per ALU input: `00` = read straight from
the register file, `10` = forward from EX/MEM, `01` = forward from MEM/WB.
EX/MEM is checked first, so if both sources would match (the same register
written by two back-to-back instructions still in flight), the more recent
value wins — which is the correct behavior. Store data goes through the same
forwarded mux (`alu_in_b_pre` feeds `pipe_exmem`'s `ex_rd2` input, not the
raw un-forwarded `ex_rd2`), so `sd` picks up a value forwarded from an
instruction still ahead of it in the pipeline correctly too — that one I was
specifically worried about getting wrong and it turned out to already be
right by construction. Both forwarding checks also exclude `rd == x0`, so
forwarding never fires for the hardwired-zero register — matches the
register file's own write protection.

<figure>
  <img src="/assets/images/foundation-core/forwarding-unit-diagram.png" alt="forwarding_unit.sv interface diagram — ex_rs1/ex_rs2/mem_rd/mem_reg_write/wb_rd/wb_reg_write in, forward_a/forward_b out">
  <figcaption><code>forwarding_unit.sv</code> — purely combinational, EX/MEM checked before MEM/WB so the more recently produced value always wins.</figcaption>
</figure>

## Hazard unit: just the load-use case

`hazard_unit.sv` only checks for one thing: a load sitting in EX whose
destination register (`ex_rd`) matches either source register of the
instruction currently in ID (`id_rs1`/`id_rs2`), gated by `ex_mem_read`. When
that happens it asserts both `stall` (freezes the PC and IF/ID) and
`id_flush` (bubbles the instruction already in ID/EX). Every other data
hazard — the much more common case of "the previous instruction's result
feeds directly into this one" — doesn't stall at all; it's handled entirely
by forwarding instead. Load-use is the one case forwarding structurally can't
fix, because the loaded value isn't available until the end of MEM, one stage
too late for the EX stage that needs it. Unlike `forwarding_unit.sv`, there's
no `x0` guard here — a load into `x0` would still (harmlessly) trigger a
stall, since a write to `x0` is a no-op anyway.

<figure>
  <img src="/assets/images/foundation-core/hazard-unit-diagram.png" alt="hazard_unit.sv interface diagram — ex_rd/ex_mem_read/id_rs1/id_rs2 in, stall/id_flush out">
  <figcaption><code>hazard_unit.sv</code> — fires only for the load-use case; general RAW hazards are forwarded, not stalled.</figcaption>
</figure>

Verilator threw `CASEINCOMPLETE` on the ALU-input forwarding muxes in
`pipeline_top.sv` the first time they went in — `forward_a`/`forward_b` are
2 bits wide but only three encodings are ever driven (`00`/`10`/`01`; `11`
never happens), and the first version of the `case` statement didn't have a
`default`, so Verilator flagged `0x3` as uncovered. Harmless in practice
(that select line was never going to be `11`), but added
`default: alu_in_a = ex_rd1;` (and the same for `alu_in_b_pre`) anyway —
cheap insurance against a stray X propagating if that ever changed, and it
matches how `control.sv` already defaults every signal before its own `case`.

**Next:** with load-use and RAW forwarding both in place, the pipeline
should be functionally complete for the current ISA subset — next week is
about what still resolves the slow way (branches, always three cycles late).

To see the load-use stall in isolation, a small standalone program — store a
value, load it back, then immediately use it, so `add` needs a value
forwarding structurally can't supply yet:

```
addi x1, x0, 7      # x1 = 7
sd   x1, 0(x0)      # mem[0] = 7
ld   x2, 0(x0)      # x2 = mem[0]        <- load ...
add  x3, x2, x1     # x3 = x2 + x1       <- ... used immediately: load-use hazard
addi x4, x0, 1      # trailing instruction, just to show the pipeline resuming
```

Traced off the actual RTL — cycle 1 is when `addi x1` is fetched, matching
the usual "instruction *k* enters IF at cycle *k*" convention. The waveform
below has the full cycle-by-cycle IF/ID/EX occupancy and every signal;
`forward_a`/`forward_b`: `00`=regfile, `01`=MEM/WB, `10`=EX/MEM. Register
writes land a cycle after WB, so `x2=7` first shows up at cycle 8, `x3=14`
at cycle 10, `x4=1` at cycle 11 — all confirmed against the final register
dump, not just asserted.

Three separate events, all real, all in this one five-instruction program:

- **Cycle 4 — store-data forwarding.** `sd` is in EX needing `x1` for its
  store data, but `addi x1` (the instruction right before it) hasn't reached
  WB yet — it's sitting in EX/MEM. `forward_b=10` picks its result up from
  there. No stall; this is the "ordinary" RAW case forwarding exists for.
- **Cycle 5 — the load-use stall.** `ex_mem_read`, `stall`, and `id_flush`
  all rise together — `ld` has just reached EX, `add` is decoding right
  behind it in ID needing `x2`, and `hazard_unit` catches the collision. All
  three drop back to 0 the very next cycle; `add` and the not-yet-fetched
  `addi x4` both hold in place for cycle 6 (that's the one stall cycle
  actually freezing fetch), then resume.
- **Cycle 7 — the load-use hazard actually resolving.** `add` reaches EX one
  cycle later than it would have unstalled, and by now `ld`'s result has
  moved on to MEM/WB — `forward_a=01` picks it up from there. `x3` ends up
  `14` (`7+7`) — correct, and it got there via forwarding, not a second
  stall.

<figure>
  <img src="/assets/images/foundation-core/load-use-stall-waveform.svg" alt="Waveform diagram of stall, id_flush, forward_a, and forward_b around a load-use hazard">
  <figcaption>All five signals from the trace above: <code>sd</code>'s store-data forward at cycle 4, the one-cycle <code>stall</code>/<code>id_flush</code> bubble at cycle 5, and <code>add</code>'s forwarded load result at cycle 7.</figcaption>
</figure>
