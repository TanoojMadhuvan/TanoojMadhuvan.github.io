---
title: KV-Cache Acceleration
slug: kv-cache
summary: A hardware accelerator for transformer KV-cache access patterns.
status: Active
color: "#C85C96"
---
Transformer inference is increasingly bottlenecked by KV-cache memory
bandwidth rather than compute. This track designs a small accelerator/memory
subsystem targeting that access pattern — REPLACE_ME: scope (e.g. a
custom-addressed scratchpad, quantized KV storage, or a prefetcher) —
integrated as a memory-mapped peripheral off the core.

**Why:** REPLACE_ME (e.g. "wanted to connect the processor-design work to a
concrete, currently-relevant ML-systems bottleneck instead of only classic
CPU microarchitecture").
