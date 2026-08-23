---
title: Out-of-Order Execution
slug: out-of-order
summary: Extending the in-order pipeline toward out-of-order issue and dynamic scheduling.
status: Active
color: "#9A6BC0"
---
The base core issues and completes instructions strictly in program order. This
track explores what it takes to relax that — REPLACE_ME: scope (e.g.
scoreboarding vs. Tomasulo's algorithm, reorder buffer, register renaming) —
and how much of it is realistic to implement and verify solo.

**Why:** REPLACE_ME (e.g. "out-of-order execution is the core idea behind
every modern high-performance processor; wanted to understand the hazard
bookkeeping — RAW/WAR/WAW, renaming — by building a (scoped-down) version").
