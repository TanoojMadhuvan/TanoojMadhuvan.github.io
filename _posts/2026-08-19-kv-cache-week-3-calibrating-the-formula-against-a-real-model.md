---
title: "Week 3: Calibrating the closed-form formula against a real model"
date: 2026-08-19
project: kv-cache-transformers
tracks: [kv-cache]
summary: "Before trusting the closed-form KV-cache formula for the policy sweep, it has to survive contact with a real model — distilgpt2, run step by step on CPU, checked against the formula on every one of 30 decode steps. Exact match, every step."
---
Last week's plan only works if the closed-form formula
(`bytes = 2 × layers × heads × head_dim × seq_len × bytes_per_element`) is
actually a correct description of what a real model does, not just a
textbook simplification that happens to look right. So before writing a
single line of the policy sweep, this week is entirely about proving that
formula against a real forward pass.

## The setup

`real/measure_kv_cache.py` runs a real Hugging Face causal LM
(`distilgpt2`, since it's small enough to run comfortably on CPU) through
30 autoregressive decode steps, and at every step:

1. Reads the config's actual `num_layers`, `num_heads`, `head_dim`.
2. Sums the real byte size of every layer's cached K and V tensors out of
   `past_key_values` — `transformers` 5.x returns a `DynamicCache`, where
   each `cache.layers[i]` exposes `.keys`/`.values` tensors directly with
   shape `[batch, heads, seq_len, head_dim]`.
3. Computes what the closed-form formula *predicts* the cache should be at
   that exact sequence length.
4. Records whether the two numbers match, byte for byte.

```python
def kv_cache_bytes(cache, dtype_bytes):
    total = 0
    for layer in cache.layers:
        total += layer.keys.numel() * dtype_bytes
        total += layer.values.numel() * dtype_bytes
    return total

def closed_form_bytes(num_layers, num_heads, head_dim, seq_len, dtype_bytes):
    return 2 * num_layers * num_heads * head_dim * seq_len * dtype_bytes
```

## Result: exact match, every step

```
model=distilgpt2 layers=6 heads=12 head_dim=64
wrote 30 steps to results/real_kv_cache_growth.csv
closed-form formula matches real measured bytes every step: True
final step: seq_len=36 kv_cache=1327104 bytes
```

Every one of the 30 steps agrees exactly — no off-by-one on sequence
length, no hidden padding, no surprise dtype mismatch. Plotting measured
against predicted makes the point visually redundant, which is exactly
what you want out of a calibration check:

<figure>
  <img src="/assets/images/kv-cache-transformers/plot-real-calibration.png" alt="Line chart of KV-cache size in KB vs sequence length in tokens, showing two lines — a solid black closed-form prediction and a dashed orange measured line from distilgpt2's real past_key_values — perfectly overlapping across the full range">
  <figcaption>Measured (distilgpt2, real <code>past_key_values</code>) vs. closed-form formula, per decode step. The dashed line is drawn on top of the solid one and is still invisible — that's the calibration succeeding.</figcaption>
</figure>

## What this does and doesn't prove

This is a **capacity** check, not a bandwidth/timing check, and it's worth
being precise about the difference so it doesn't get overstated later. It
proves the formula correctly predicts *how many bytes* a real model's
KV-cache occupies at a given sequence length — which is exactly what the
SRAM/HBM capacity crossover in the planned sweep depends on. It says
nothing about real memory bandwidth, real access latency, or real
bandwidth-bound behavior once the cache overflows on-chip memory, because
this ran on CPU — there's no local CUDA GPU available yet. That GPU
re-run, once TACC or similar access is sorted, is what would validate the
*timing* side; this week only validates the *capacity* side. Both matter,
but they're not the same claim, and the README tracks them as two
separate, still-open items rather than one "done."

## Next

With the formula trustworthy for capacity accounting, the synthetic sweep
no longer needs to run a real model per configuration — it can generate
the same access-pattern shape directly and cheaply, across many context
lengths, SRAM budgets, and all four mitigation policies at once.
