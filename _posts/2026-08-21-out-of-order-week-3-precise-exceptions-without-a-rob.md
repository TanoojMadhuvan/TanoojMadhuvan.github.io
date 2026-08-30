---
title: "Week 3: Precise exceptions without a reorder buffer"
date: 2026-08-21
project: nodeflow
tracks: [out-of-order]
summary: "This is the problem HPS's own papers don't fully solve, which is exactly why Smith & Pleszkun's precise-interrupts paper got pulled in. The answer turns out to already be sitting inside last week's engine: in-order retirement gives precise state for free — the only new work is making a fault use that same machinery."
---
Tomasulo-style machines usually reach for an explicit reorder buffer to get
precise exception state — the ROB *is* the mechanism that lets younger,
already-completed instructions get discarded without corrupting
architectural state. HPS has no ROB. Smith & Pleszkun's "Implementation of
Precise Interrupts in Pipelined Processors" is the paper this project is
leaning on to answer what replaces it here.

## The question

If a faulting instruction has independent, faster instructions dispatched
right behind it, those can finish executing before the fault is even
discovered. Precise exception semantics require the machine to behave as
if none of them ran — architectural state must reflect exactly the
instructions before the fault, and none after.

## The mechanism (or: the one already built)

The answer is almost embarrassingly small given what's already in place.
Retirement has been strictly in-order since week 1 — nothing commits to
architectural state until it's the single oldest node in the table. That
already means no instruction after a fault could have corrupted anything,
*provided* the fault is caught before that faulting instruction retires.

So a faulting op (currently just DIV by zero) doesn't compute a result at
all — it marks itself and waits for its normal turn like anything else.
When it becomes the oldest node and is about to retire, instead of
committing, it traps: every other in-flight node gets flushed, including
any that raced ahead and already completed, and dispatch stops. There's no
separate recovery mechanism to design because in-order retirement was
already doing the hard part.

## Trace: a fast op that finishes early and still gets discarded

```
cycle 0: node seq=0 dispatches (ADD)
cycle 1: node seq=0 (ADD) issues
cycle 1: node seq=1 dispatches (DIV)
cycle 2: node seq=0 (ADD) completes, result=20
cycle 2: node seq=0 retires, R4 = 20
cycle 2: node seq=1 (DIV) issues
cycle 2: node seq=2 dispatches (ADD)
cycle 3: node seq=2 (ADD) issues
cycle 4: node seq=2 (ADD) completes, result=2
cycle 6: node seq=1 (DIV) divide-by-zero -- will trap at retire
cycle 6: node seq=1 traps at retire -- flushing all in-flight state
```

Node seq=2 completes at cycle 4 with a perfectly correct result — and
still never retires, because it's younger than the fault discovered two
cycles later. Final architectural state has R4 (older than the fault)
committed and R5/R6 (the fault's own destination, and the younger
instruction) both at their untouched initial values. That's precise state,
and the golden check that matters most here isn't "does the fault get
detected" — it's "does the already-completed younger instruction still get
thrown away."

## Next

Exceptions only had to discard register writes. Memory needs the same
discipline extended to stores — which also turns out to matter for a
completely different reason: knowing when a load is even allowed to read.
