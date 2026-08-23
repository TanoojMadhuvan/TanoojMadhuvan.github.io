---
title: "Week 6: Closing the loop — oracle, synthesis, timing, power"
date: 2026-02-16
project: foundation-core
tracks: [verification, ppa-analysis]
summary: "The C++ oracle catches a real pipeline bug, then Yosys and OpenSTA give the first area, timing, and power numbers — dmem dominates all three."
---
Last week of the Foundation Core arc. Four things happened:

- Used the C++ simulator as a verification oracle, not just a demo tool
- That caught a real bug in the pipelined core
- Got first synthesis numbers out of Yosys
- Got first timing + power numbers out of OpenSTA

## The oracle catches a real bug

`single_cycle.cpp` (388 lines) is a cycle-accurate model of `rtl/top.sv` —
same stages, same instruction set. Used it as a second, independent
implementation to diff against the RTL's final register/memory state.

- Neither `tb/top_tb.cpp` nor `pipeline/tb/pipeline_tb.cpp` automates this
  diff yet — both just run 10,000 cycles and dump whatever's nonzero. The
  comparison is still me reading two terminal outputs side by side.
- Running the canonical `addi x1,x1,-1` loop program (from `INSTRUCTIONS.md`)
  through both cores:
  - Single-cycle core: matched the oracle exactly (`x2=55`)
  - Pipelined core: **did not** — came out `x2=45`

**Root cause:** the pipelined register file reads combinationally off the
*not-yet-updated* array. If a producer and consumer are exactly 3
instructions apart (this loop's exact spacing), the producer's write and the
consumer's read land in the same cycle — `forwarding_unit` can't catch it,
because the producer has already fully retired past the two points it
checks.

**Fix:** new module `wb_bypass.sv`, sitting between `regfile`'s outputs and
`pipe_idex`'s inputs. Deliberately *not* folded into `regfile.sv` — that was
the first attempt, and it broke the single-cycle core with a genuine
combinational loop (its write data is derived combinationally from the same
reads, same cycle, for instructions like `addi x1,x1,-1`).

After the fix: both cores match the oracle exactly. This is also proof the
oracle-diff approach actually works — this bug only surfaced because the
RTL's output was being checked against the C++ model's, by hand, for this
program.

## First synthesis pass

Ran both flows through **Yosys** (compiles RTL into a gate-level netlist
using a real standard-cell library):

- `synth.ys` — generic, technology-independent
- `synth_nangate.ys` — mapped to Nangate45 (a 45nm open standard-cell
  library) via OpenROAD-flow-scripts

Two warnings, both harmless, both pointing at the same line
(`assign reg_out = rf0.regs[3];` — a debug hook into the register file's
internal array):

```
rtl/top.sv:96: Identifier `\rf0.regs' is implicitly declared.
rtl/top.sv:96: Range select out of bounds on signal `\rf0.regs'.
```

## Area: dmem is 96% of the chip

| Module | Area (µm²) |
|---|---|
| `dmem` | 488,938.06 |
| `regfile` | 18,845.57 |
| `alu` | 909.72 |
| `immgen` | 77.14 |
| `imem` | 22.34 |
| `control` | 10.91 |
| `alu_control` | 13.03 |
| **Total** | **510,014.30** |

**`dmem` = 95.87% of the chip.** Why:

- Yosys never inferred `dmem`'s array as a real memory macro (`Number of
  memories: 0` in the log)
- It got flattened into 1,024 individual flip-flops instead — each one a
  full standard cell, not a dense SRAM bitcell
- `imem`'s tiny number (22 µm²) isn't comparable — it has no write port, so
  Yosys just folds it into a constant lookup table for whatever's currently
  in `hex.txt`. Not a real ROM measurement, just an artifact of a short
  program.

<figure>
  <img src="/assets/images/foundation-core/chip-placement-openroad-gui.png" alt="OpenROAD GUI showing the placed die, color-coded by submodule: dmem in green covering almost the entire die, regfile in yellow as a small block in the corner, alu in magenta barely visible inside it">
  <figcaption>Live from <strong>OpenROAD</strong> (floorplanning + placement + routing tool) after global placement finished (199,021 cells) — colored by submodule: <code>dmem</code> in green, <code>regfile</code> in yellow, <code>alu</code> in magenta. No routing or clock tree yet, just where the cells landed — but it makes the 96% number visual instead of just a table row. (The Inspector panel on the right is mid-click on one <code>alu0</code> inverter, not anything load-bearing.)</figcaption>
</figure>

## Timing + power: first real numbers

Ran **OpenSTA** (static timing analysis + power estimation, reads the
synthesized netlist + a `.sdc` clock constraint file) against the Nangate45
netlist and the existing 10ns/100MHz clock target.

**Timing — badly violated:**
- Worst negative slack: **-200.31 ns** against a 10ns budget
- This core would need to run at ~4.75 MHz, not 100 MHz, to actually meet
  its own constraint as synthesized
- Critical path: `imem` → `regfile` → `alu` → `dmem`, all in one cycle
  (expected for a single-cycle core)
- One gate inside `dmem` alone eats 128 ns of that 210 ns path — a
  write-enable gate fanning out to all 1,024 flip-flops at once. Same root
  cause as the area problem: a real SRAM macro's decode logic is internal
  and pre-characterized, so it never shows up as one gate with massive
  fanout.
- Hold timing is fine (+0.10 ns, met)

**Power — 72.1 mW total:**
- 52.5 mW internal, 10.7 mW switching, 8.8 mW leakage
- 73.2% of it is `dmem`'s 1,024 flip-flops
- Same story again: dominated by modeling an 8KB memory as flip-flops, not
  by actual datapath logic
- **Not from running an actual program** — this is OpenSTA's flat default
  (`set_power_activity -activity 0.5`, i.e. "assume every input toggles 50%
  of the time"), not real switching activity from simulating a test program.
  A per-program number would need a VCD dumped from Verilator while running
  one, fed into OpenSTA instead of the flat guess.

Full reports: `synth/sta_log.txt`.

## Next

- **Swap `dmem`/`imem` onto real SRAM macros.** OpenROAD-flow-scripts
  already ships pre-generated `fakeram45` macros (pre-characterized
  area/timing/power) — other example designs there use them the same way.
  `fakeram45_1024x32` is an exact match for `imem`. `dmem` (1024×64) would
  need two `fakeram45_512x64` banked on one address bit. Open question:
  how to black-box a macro into an RTL module still shared with behavioral
  simulation, without breaking the single-cycle/pipelined sims.
- **Cheaper first step: just shrink `dmem`'s depth.** Nothing in the ISA
  requires 1,024 entries — test programs use a fraction of that. Cutting to
  256 entries would cut flip-flop count (and area, and that fanout gate) to
  roughly a quarter, with a one-line RTL change. Doesn't fix the underlying
  flip-flop-vs-SRAM density gap, but it's a real, immediate win stackable
  with the macro swap above.

[screenshot/waveform reference — none yet; `synth/synth_nangate_log.txt` and
`synth/sta_log.txt` are in the repo if a log excerpt is more useful]
