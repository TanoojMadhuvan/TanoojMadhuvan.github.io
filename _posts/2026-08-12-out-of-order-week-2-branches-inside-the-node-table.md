---
title: "Week 2: Branches inside the node-table model"
date: 2026-08-12
project: nodeflow
tracks: [out-of-order]
summary: "MICRO-18 doesn't say what happens to a node table on a misprediction. The answer that falls out of an already-in-order-retire engine: checkpoint the register-tag table at dispatch, predict not-taken, and squash everything younger if that guess was wrong."
---
The intro HPS paper describes tag matching for straight-line dependencies
in detail but waves at branches — that's exactly the kind of gap "Critical
Issues Regarding HPS" exists to fill. This week is about working out a
concrete answer inside last week's node table, not just reading about one.

## The question

Once the node table is dispatching down a predicted path, a misprediction
has to undo exactly the right set of nodes and leave the register-tag
table in a state consistent with the *correct* path — not the speculative
one.

## The mechanism

Predicting is trivial here: always predict not-taken, so dispatch just
keeps going in program order and no speculative redirection logic is
needed. All the real work is in recovery:

1. A branch snapshots the entire register-tag table into itself at
   dispatch time.
2. When it resolves (same "complete" phase every other op uses), if it's
   actually taken, every node younger than it — by sequence number, not
   physical table position — gets invalidated, the register-tag table is
   restored from the branch's own snapshot, and dispatch is redirected to
   the branch target.

The reason this is *safe* rather than merely *convenient* is last week's
invariant: retirement is strictly in program order, so nothing younger
than an unretired branch could possibly have committed anything yet.
Squashing them is just freeing table slots, not undoing architectural
state.

## Trace: a misprediction in practice

```
cycle 0: node seq=0 dispatches (ADD)
cycle 1: node seq=0 (ADD) issues
cycle 1: node seq=1 dispatches (BEQ)
cycle 2: node seq=0 (ADD) completes, result=10
cycle 2: node seq=0 retires, R2 = 10
cycle 2: node seq=1 (BEQ) issues
cycle 2: node seq=2 dispatches (ADD)
cycle 3: node seq=1 (BEQ) resolves taken (misprediction)
cycle 3: squashed younger nodes, redirecting dispatch to instruction 3
cycle 3: node seq=1 retires (branch)
cycle 3: node seq=3 dispatches (SUB)
```

Node seq=2 dispatches at cycle 2, right behind the branch — and then just
vanishes. It never issues, never completes, never retires. That's the
squash working as intended.

The trickier golden check isn't the squash itself, it's making sure the
checkpoint restore is actually correct: a wrong-path instruction can claim
to be the producer of some register, and if that claim survives the
squash, a later correct-path instruction reading that register would wait
on a tag that will never broadcast again — a permanent stall. Restoring
the entire register-tag table from the branch's own dispatch-time snapshot
is what prevents that.

## Next

Branches only had to interact with the in-order-retire invariant this
week. Exceptions need it to do more: precise state has to survive not just
control-flow squashes but an instruction actually faulting mid-flight.
