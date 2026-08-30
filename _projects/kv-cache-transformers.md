---
title: "KV-Cache & Transformers"
slug: kv-cache-transformers
index: "02"
date_range: "Jul 21, 2026–ongoing"
icon: 09-database
subtitle: "Learning Transformer Inference on Real Hardware"
summary: "Hands-on transformer inference and KV-cache profiling on a real GPU — the actual workload the KV-Cache Acceleration track is meant to be built around."
description: "Learning how transformer inference and the KV-cache actually behave by running real models on real GPU hardware and profiling memory growth during autoregressive decode, instead of simulating an assumed access pattern — grounded in background reading including the original Transformer paper (Vaswani et al., \"Attention Is All You Need\")."
tags: [Python, PyTorch, CUDA, Transformers]
tracks: [kv-cache]
---
Replaces an earlier plan to build a CPU-side cache-hierarchy simulator with
something more direct: actually running transformer inference on real GPU
hardware and measuring how the KV-cache behaves, instead of simulating an
assumed access pattern. The [KV-Cache Acceleration](/tracks/kv-cache/)
track's eventual hardware accelerator only makes sense if it's built for the
real workload — this project starts by learning that workload firsthand,
on real hardware, before designing anything to accelerate it.

Background reading includes the original Transformer paper (Vaswani et al.,
["Attention Is All You Need"](https://arxiv.org/pdf/1706.03762)) for context
on why the Q/K/V attention mechanism produces the specific memory-growth
pattern KV-cache exists to handle — read for that context, not as a spec
being reimplemented from scratch.
