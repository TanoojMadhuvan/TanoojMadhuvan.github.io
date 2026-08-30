---
title: "Week 1: Why KV-cache, and why this needs a real GPU"
date: 2026-07-21
project: kv-cache-transformers
tracks: [kv-cache]
summary: "Dropping the plan to simulate KV-cache access patterns on the CPU side, in favor of actually running transformer inference on a real GPU and measuring what the cache does — starting with why the paper that defines attention also explains why this memory-growth problem exists at all."
---
Changing direction before writing any more code. The original plan for this
project was a CPU-side cache-hierarchy simulator — configurable, MESI
coherence, the works — *motivated* by KV-cache access patterns without
actually running a transformer at all. That's backwards for what this track
is supposed to answer. If the eventual goal is a hardware accelerator for
KV-cache traffic, it needs to be built against the *real* access pattern,
not an assumed one. So: real models, real GPU, real profiling, starting now.

## Why KV-cache exists at all

Background reading starts at the source: Vaswani et al.,
["Attention Is All You Need"](https://arxiv.org/pdf/1706.03762) — the paper
that defines the Q/K/V attention mechanism every transformer since has been
built on. Reading it for one specific reason: understanding *why* decoding
one token at a time in an autoregressive model produces a memory footprint
that keeps growing, rather than treating "KV-cache is a bottleneck" as a
given fact to accelerate around.

The shape of it, in short — every generated token needs to attend back over
every previous token's Key and Value vectors. Recomputing those K/V vectors
from scratch for the entire growing sequence on every single new token would
be wasteful, so real inference systems cache each token's K/V vectors the
first time they're computed and reuse them for every later step. That cache
grows by one token's worth of K/V vectors every decode step, for the entire
length of the generation — which is the actual object this track is
eventually meant to accelerate. This is background context for *why* the
cache grows the way it does, not a spec being reimplemented from the paper —
the actual K/V/Q math is what an existing framework (PyTorch, Hugging Face
`transformers`) already runs correctly; the point here is measuring its
real behavior, not rebuilding it.

## Why this needs to move to a real GPU

The previous plan's flaw, stated plainly: a simulator built around an
*assumed* KV-cache access pattern is only as good as that assumption. The
actual pattern — how fast the cache grows, what the read/write traffic
looks like per layer, per attention head, across a real batch of real
prompts — is something a real model produces, not something to guess at from
a paper. That means this phase needs actual GPU hardware: running a real
(small, to start) open model, generating text autoregressively, and
profiling what its KV-cache actually does during that process — memory
growth per decode step, bandwidth pressure, how it scales with sequence
length and batch size.

## What "done" looks like for this phase

Not a finished accelerator design — a real, measured baseline:

- A small open model running real autoregressive decode on a GPU.
- Actual profiling of KV-cache memory growth and access traffic during that
  decode, not an estimate.
- That real data becoming the actual target for the
  [KV-Cache Acceleration](/tracks/kv-cache/) track's hardware work, instead
  of a hardware design built against an assumption.

**Next:** get GPU access sorted, pick a specific model and inference
framework, and get one real decode run profiled end to end.
