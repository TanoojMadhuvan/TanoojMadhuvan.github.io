---
title: "Week 1: Choosing HPS over Tomasulo"
date: 2026-07-28
project: nodeflow
tracks: [out-of-order]
summary: "Going out-of-order with Patt/Hwu/Shebanow's HPS restricted-dataflow model instead of Tomasulo's reservation stations — and a working C++ node-table prototype to prove it."
---
## The question

The in-order pipeline stalls on any hazard it can't forward around. Going
out-of-order means letting independent instructions behind a stalled one
execute anyway — which means tracking dependencies dynamically instead of
relying on the fixed pipeline structure.

## Options considered

- **Scoreboarding** — simplest option, no register renaming, but doesn't
  handle WAR/WAW hazards cleanly.
- **Tomasulo's algorithm** — reservation stations attached to each
  functional unit, a common data bus for broadcasting results, and (in most
  modern treatments) an explicit reorder buffer for precise state. This is
  what nearly every course, textbook, and hobbyist core defaults to.
- **HPS (Patt, Hwu, Shebanow, MICRO-18 1985)** — models the program as a
  restricted dataflow graph. Instructions become nodes in a central node
  table; dependencies are resolved by tag matching (a completing node
  broadcasts its own identity as a tag, and any node waiting on that tag
  captures the result) rather than by reservation stations distributed
  across functional units.

## Decision

Going with HPS. Two reasons:

1. **It's a different lineage.** Tomasulo/ROB is the default answer for
   out-of-order execution in almost every reference — building it again
   wouldn't teach me much I couldn't already predict. HPS's restricted
   dataflow graph is a genuinely different way of expressing the same
   problem, and Tomasulo is still useful here as the contrast that makes
   HPS's design choices legible, not as something to implement.
2. **The intro paper leaves real gaps.** MICRO-18's introduction doesn't
   fully specify exceptions, branches, or memory disambiguation inside the
   dataflow model — the way an explicit reorder buffer gets those almost for
   free in Tomasulo. Two follow-ups fill in the rest of what's needed:
   "Critical Issues Regarding HPS, A High Performance Microarchitecture"
   (Patt, Hwu, Shebanow, 1985) covers exceptions, branches, and memory
   disambiguation directly, and HPSm (Patt, Melvin, Hwu, Shebanow, 1986)
   gives a scaled-down, concretely-specified version of HPS useful for
   resolving ambiguities by example. Precise exception recovery is still an
   open problem for a no-ROB dataflow machine even after those two, so
   Smith & Pleszkun's "Implementation of Precise Interrupts in Pipelined
   Processors" (1985) is being pulled in specifically to solve that piece
   before any RTL exists — not repeated after.

Before touching hardware, the plan is: (1) a C++ functional/cycle-level
prototype of the node-table scheduling mechanism, to prove the dependency
resolution is actually correct and work out the exception/memory-disambiguation
design on paper-fast iteration instead of in a waveform viewer, (2)
SystemVerilog RTL on top of [Foundation Core](/projects/foundation-core/)'s
existing pipeline, verified with UVM, and (3) real synthesis and PPA numbers
through the same Yosys + OpenROAD + OpenSTA flow already used for SysMAC and
Foundation Core.

## C++: the node table entry (from the working prototype)

Phase 1's prototype is already running. Each node tracks two operands that
are either an already-available value or a tag naming the in-flight node
that will produce it:

```cpp
struct Node {
    bool valid = false;
    int seq = -1;          // doubles as this node's tag once it produces a result
    Op op = Op::ADD;
    int dest_reg = -1;

    bool src1_ready = false;
    int32_t src1_val = 0;
    int src1_tag = -1;     // producing node's seq, -1 if no producer pending

    bool src2_ready = false;
    int32_t src2_val = 0;
    int src2_tag = -1;

    bool issued = false;
    int remaining_latency = 0;
    bool result_ready = false;
    int32_t result = 0;
};
```

Every cycle runs complete → retire → issue → dispatch: a completing node
broadcasts its own sequence number as a tag, only the single oldest valid
node may retire (execution is out-of-order, retirement stays strictly
in-order), any node with both operands ready can issue regardless of its
position in the table, and a new instruction dispatches by snapshotting its
operands against whichever node currently owns each source register.

A trace from the demo driver shows exactly the property a fixed pipeline
can't express — an independent ADD dispatched right behind a slow MUL
completes first, but still retires after it:

```
cycle 0: node seq=0 dispatches (MUL -> R4)
cycle 1: node seq=0 (MUL) issues
cycle 1: node seq=1 dispatches (ADD -> R5)
cycle 2: node seq=1 (ADD) issues
cycle 3: node seq=1 (ADD) completes, result=11
cycle 4: node seq=0 (MUL) completes, result=6
cycle 4: node seq=0 retires, R4 = 6
cycle 5: node seq=1 retires, R5 = 11
```

Three golden checks (a RAW chain, this reordering case, and a WAW/WAR case
that would break if operands were snapshotted at the wrong time) all pass
against hand-derived expected register values. Code's at
[tanooj-comp-arch/NodeFlow](https://github.com/tanooj-comp-arch/NodeFlow).

## Next

Branches and precise exceptions inside the node-table model — the two
pieces the intro paper hand-waves and the Critical Issues / Smith &
Pleszkun papers actually have to earn.
