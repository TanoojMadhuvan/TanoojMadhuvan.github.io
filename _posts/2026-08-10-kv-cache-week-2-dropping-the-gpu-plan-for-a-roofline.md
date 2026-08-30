---
title: "Week 2: Dropping the GPU-profiling plan for a roofline model"
date: 2026-08-10
project: kv-cache-transformers
tracks: [kv-cache]
summary: "Real GPU access still hasn't come through, so instead of waiting on it, the project pivots to a roofline-style memory-hierarchy model: a closed-form KV-cache formula swept across mitigation policies, validated against a real model on CPU first — reframed and renamed KVRoof."
---
Week 1 ended with a plan to move onto real GPU hardware and profile actual
KV-cache traffic instead of reasoning about it abstractly. That GPU access
(TACC or equivalent) hasn't materialized yet, and waiting on it indefinitely
isn't a plan — so this week is a second pivot, not a continuation: instead
of profiling a real model's KV-cache on real hardware, model the
memory-hierarchy problem directly with a closed-form formula, swept across
mitigation techniques, and validate that formula against something real
that *doesn't* require a GPU — a CPU inference run. The project gets a new
name to match: **KVRoof**.

## What the project is actually asking now

For a transformer *inference* accelerator, the differentiating hard problem
usually isn't the compute array — it's attention, specifically the
KV-cache: memory that grows every generated token and has to be read in
full on every subsequent step. The question worth answering isn't just
"how big does it get," it's: given a fixed on-chip SRAM budget, where's the
cliff — the context length at which the cache stops fitting on-chip and
forces the accelerator into HBM-bandwidth-bound territory — and how much do
real, published mitigation techniques delay it?

Four policies, picked because they're grounded in real papers rather than
invented for this project:

- **Baseline** — full-precision KV-cache, no eviction, no width reduction.
- **GQA / MQA** ([Grouped-Query Attention](https://arxiv.org/abs/2305.13245),
  [Multi-Query Attention](https://arxiv.org/abs/1911.02150)) — shares K/V
  across attention heads, shrinking the cache at the architecture level.
- **[H2O](https://arxiv.org/abs/2306.14048)** (Heavy-Hitter Oracle) —
  evicts low-attention-score tokens once the cache exceeds a budget.
- **[KVQuant](https://arxiv.org/abs/2401.18079)** — ultra-low-bit
  quantization of the cache itself.

## Why a closed-form model instead of running real inference for the sweep

The sweep needs to vary context length, head count, and bit-width across
many configurations. Two things rule out doing that with real models:

1. Frameworks like Hugging Face `transformers` don't expose an on-chip
   SRAM boundary at all — `past_key_values` just lives in whatever RAM/VRAM
   the host has. "Did the cache overflow a 40MB budget at token 512" isn't
   a number that exists in that stack; it only exists in a memory-hierarchy
   model built specifically to track it.
2. The thing that changes across the sweep — context length, GQA group
   size, eviction budget, quantization width — only affects the
   byte-accounting of the access pattern, not its shape: every decode step
   reads the whole existing cache and appends one new K/V pair, regardless
   of which real model produces it. Re-deriving that fixed shape by running
   dozens of real forward passes buys nothing a formula doesn't already
   give for free.

So: **closed-form formula for the sweep, but only once it's confirmed
against something real.**

## Back-of-envelope, before writing any simulator code

Before building anything, it's worth checking this problem is even worth
building for. Using the textbook formula
(`bytes = 2 × layers × heads × head_dim × seq_len × bytes_per_element`,
fp16) against a flat 40MB SRAM budget — a plausible on-chip cache size —
across a few real published model shapes:

| Model scale | Bytes per token (K+V, fp16) | Tokens until 40MB SRAM overflows |
|---|---|---|
| 1.3B-class (24L / 16H / d128) | 192 KB | 213 |
| 7B-class (32L / 32H / d128) | 512 KB | 80 |
| 13B-class (40L / 40H / d128) | 800 KB | 51 |
| 70B-class if it used plain MHA (80L / 64H / d128) | 2.5 MB | 16 |

Even a 7B-class model with standard multi-head attention blows past a 40MB
on-chip budget in **80 tokens** — a couple of sentences of generation. At
70B scale under plain MHA it's 16 tokens — which is also exactly why real
70B-class models (Llama-2-70B included) don't ship with plain MHA; they use
GQA specifically to survive this. That's not a coincidence this project is
discovering — it's the reason GQA exists, showing up in the same formula
this project is about to sweep.

## Next

- Scaffold the repo structure (`sim/`, `golden/`, `real/`, `results/`) and
  pick the calibration target: a small model, running on CPU since GPU
  access isn't available yet, checked step-by-step against the closed-form
  formula above.
- If that calibration holds, the formula is trustworthy enough to build the
  actual policy sweep on top of without needing a GPU at all for the
  *shape* of the answer — only for later, real bandwidth/timing numbers.
