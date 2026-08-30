---
title: "KV-Cache & Transformers"
slug: kv-cache-transformers
index: "02"
date_range: "Jul 21, 2026–ongoing"
icon: 09-database
subtitle: "KVRoof: A Roofline Model for KV-Cache Memory Pressure"
summary: "A roofline-style model of when a transformer's KV-cache outgrows on-chip SRAM under continuous decode, and how much GQA/MQA, H2O eviction, and KVQuant quantization each buy back before that cliff — the closed-form formula calibrated against a real Hugging Face model first."
description: "Models the KV-cache memory-hierarchy cliff directly: given a fixed on-chip SRAM budget, where does an accelerator's KV-cache stop fitting on-chip and force it into HBM-bandwidth-bound territory, and how much do GQA/MQA, H2O eviction, and KVQuant quantization each delay that cliff. The closed-form formula behind the sweep is calibrated against real Hugging Face model output before being trusted, rather than assumed."
tags: [Python, PyTorch, Transformers, Matplotlib]
tracks: [kv-cache]
---
Started as a plan to profile real transformer inference on a real GPU;
pivoted again once that GPU access didn't materialize in time. **KVRoof**
models the same underlying question — what does a growing KV-cache do to
an accelerator's memory hierarchy — as a roofline-style closed-form sweep
instead: given a fixed on-chip SRAM budget, find the context length at
which the cache stops fitting on-chip, and measure how much each of three
real, published mitigation techniques (GQA/MQA, H2O eviction, KVQuant
quantization) delays that cliff versus a full-precision baseline.

The sweep itself never runs a real model — it generates the same
decode-step access-pattern shape a real model produces (read the whole
cache, append one new K/V pair) directly from a closed-form formula. That
formula is calibrated first, though: `real/measure_kv_cache.py` runs an
actual Hugging Face model step by step and checks the formula against its
real `past_key_values` byte sizes before any synthetic sweep is trusted to
build on top of it.

Background reading includes the original Transformer paper (Vaswani et al.,
["Attention Is All You Need"](https://arxiv.org/pdf/1706.03762)) for why
the Q/K/V attention mechanism produces this specific memory-growth pattern
in the first place, plus the papers behind each mitigation technique
([GQA](https://arxiv.org/abs/2305.13245),
[MQA](https://arxiv.org/abs/1911.02150),
[H2O](https://arxiv.org/abs/2306.14048),
[KVQuant](https://arxiv.org/abs/2401.18079)).
