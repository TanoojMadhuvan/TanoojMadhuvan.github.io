---
title: KV-Cache Acceleration
slug: kv-cache
summary: A hardware accelerator for transformer KV-cache access patterns.
status: Active
color: "#C85C96"
---
Transformer inference is increasingly bottlenecked by KV-cache memory
bandwidth rather than compute. This track's end goal is a small
accelerator/memory subsystem targeting that access pattern, integrated as a
memory-mapped peripheral off the core — but that design only means something
if it targets the *real* access pattern. Current phase, in
[KV-Cache & Transformers](/projects/kv-cache-transformers/): run real
transformer inference on a real GPU and profile what the cache actually
does — growth per decode step, bandwidth, how it scales — before designing
any hardware around it.

**Why:** wanted to connect the processor-design work on this site to a
concrete, currently-relevant ML-systems bottleneck instead of only classic
CPU microarchitecture — and to design the eventual accelerator against a
measured workload, not an assumed one.
