---
title: "Week 4: Memory disambiguation — and a bug the golden checks caught"
date: 2026-08-30
project: nodeflow
tracks: [out-of-order]
summary: "The last of the three gaps the intro HPS paper leaves open. Also: a real bug in the first version, caught only because a golden check got rewritten to stop letting a coincidental zero hide it."
---
Registers have a clean answer to "who produces this value" because
register names are static — the node table already tracks that via the
register-tag table. Memory addresses are computed at runtime, so "does
this load alias an earlier, unretired store" isn't knowable until the
address itself is.

## The question

A load must never read a stale value out from under an older, in-flight
store to the same address — but it also shouldn't stall on stores it has
nothing to do with.

## The mechanism

Conservative by design, not the more aggressive speculate-and-recover
scheme the Critical Issues paper also describes: a load may only issue
once every older, still-in-flight store's address is known. If none of
them alias, the load reads memory directly. If one does, the load
forwards that store's data value directly instead — and the nice part is
this reuses the exact same tag-broadcast machinery every ordinary
register dependency already goes through, rather than needing a separate
forwarding path. Stores, like register writes, only take effect (to
memory, at a specific address) at retire — so they're automatically
ordered correctly relative to each other regardless of which one finishes
*executing* first.

## Trace: a load waiting on an address, then forwarding

```
cycle 0: node seq=0 dispatches (MUL)
cycle 1: node seq=0 (MUL) issues
cycle 1: node seq=1 dispatches (STORE)
cycle 2: node seq=2 dispatches (LOAD)
cycle 4: node seq=0 (MUL) completes, result=6
cycle 4: node seq=0 retires, R2 = 6
cycle 4: node seq=1 (STORE) issues
cycle 5: node seq=1 (STORE) completes, result=42
cycle 5: node seq=1 retires, mem[6] = 42
cycle 5: node seq=2 (LOAD) issues
cycle 7: node seq=2 (LOAD) completes, result=42
cycle 7: node seq=2 retires, R4 = 42
```

The store and the load both target an address computed by a slow MUL.
Notice the store doesn't issue until cycle 4 — the same cycle the MUL's
address finally resolves — and the load doesn't issue until cycle 5, one
cycle after the store. Neither raced ahead of what it could actually know
yet.

## The bug

That "doesn't issue until cycle 4" line wasn't true in the first version.
STORE's issue condition checked that its *data* operand was ready but
never checked that its *address* was — LOAD's readiness correctly chains
through the memory-hazard check (which already requires a known address),
but the equivalent check for STORE was just missing. The first run of this
exact demo didn't show it, because the address in that first version was
`0 * anything = 0`, which is also the field's zero-initialized default —
the store "worked" for the wrong reason, silently.

It only surfaced because the golden check got rewritten to use a
non-zero, non-default address and to assert that an *unrelated* memory
location stayed untouched. That's the whole reason for writing golden
checks against specific hand-derived values instead of just eyeballing
whether a trace looks plausible — a plausible-looking trace and a coincidental
zero look identical until something is deliberately chosen not to be zero.

## Where this leaves Phase 1

Branches, precise exceptions, and memory disambiguation — the three things
the MICRO-18 intro paper didn't spec out — now all exist as working code
with golden checks, on top of the tag-matching engine from week 1. Real
RV32IM decode (replacing the synthetic op format) and the SystemVerilog RTL
phase on top of Foundation Core are next, but not started yet — see the
[repo's status list](https://github.com/tanooj-comp-arch/NodeFlow#status)
for exactly what's done versus still open.
