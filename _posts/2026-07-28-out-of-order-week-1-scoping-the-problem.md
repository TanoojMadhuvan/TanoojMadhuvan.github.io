---
title: "Week 1: Scoping out-of-order execution"
date: 2026-07-28
project: ooo-core
tracks: [out-of-order]
summary: "Deciding how much of classic OoO (scoreboarding vs. Tomasulo, ROB, renaming) is realistic to build."
---
REPLACE_ME: replace with your actual write-up.

## The question

The in-order pipeline stalls on any hazard it can't forward around. Going
out-of-order means letting independent instructions behind a stalled one
issue anyway — which means tracking hazards dynamically instead of relying on
the fixed pipeline structure.

## Options considered

REPLACE_ME:
- Scoreboarding — simpler, no register renaming, but doesn't handle WAR/WAW as
  cleanly.
- Tomasulo's algorithm — reservation stations + a common data bus, handles
  WAR/WAW via renaming.

## SystemVerilog: reservation station entry (sketch)

```systemverilog
typedef struct packed {
    logic        busy;
    logic [4:0]  op;
    logic [31:0] vj, vk;   // operand values (if ready)
    logic [3:0]  qj, qk;   // producing RS tag (if not ready)
    logic [3:0]  dest_tag;
} rs_entry_t;
```

## Decision

REPLACE_ME: which approach you're starting with and why.
