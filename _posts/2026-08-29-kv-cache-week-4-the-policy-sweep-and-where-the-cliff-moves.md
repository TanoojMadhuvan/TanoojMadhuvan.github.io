---
title: "Week 4: The policy sweep, and where the cliff actually moves"
date: 2026-08-29
project: kv-cache-transformers
tracks: [kv-cache]
summary: "With the closed-form formula calibrated, this week builds the actual roofline sweep — a two-tier SRAM/HBM memory model and all four mitigation policies (baseline, GQA, KVQuant, H2O) — plus a golden cross-check and the first comparison plots. Baseline hits the SRAM cliff at decode step 80; H2O never hits it at all."
---
Formula's calibrated, so this week is the actual sweep: a synthetic
decode-step generator, a two-tier memory model, all four mitigation
policies running against it, a golden cross-check, and the first
comparison plots.

## The pieces

**`sim/workload.py`** generates the same access-pattern *shape* the real
calibration run confirmed last week — every decode step reads the entire
existing cache and appends one new K/V pair — without running any real
model weights:

```python
def generate_decode_steps(cfg, num_steps, start_seq_len=0):
    per_token = cfg.per_token_kv_bytes
    for t in range(num_steps):
        seq_len_before = start_seq_len + t
        seq_len_after = seq_len_before + 1
        yield DecodeStep(
            step=t,
            seq_len=seq_len_after,
            bytes_read=per_token * seq_len_before,
            bytes_written=per_token,
            total_cache_bytes=per_token * seq_len_after,
        )
```

**`sim/memory.py`** is deliberately the simplest version of a two-tier
model: a binary fit/no-fit check per step. While the live cache fits in the
SRAM budget, that step's read/write is serviced at SRAM bandwidth;
once it doesn't, the whole step falls to HBM bandwidth instead. It's not a
partial-LRU cache — that nuance is exactly what H2O eviction is for,
modeled separately as its own policy rather than folded into the memory
tier.

**`sim/policies.py`** implements all four. GQA/MQA and KVQuant turn out to
be pure config transforms — they change how many bytes a token costs
without changing the access pattern's shape, so they reuse
`generate_decode_steps` directly:

```python
def gqa(cfg, kv_heads):
    return replace(cfg, num_heads=kv_heads)   # only K/V heads shrink

def kvquant(cfg, bits):
    return replace(cfg, bytes_per_element=bits / 8)
```

H2O is different — it changes *which* tokens are even resident, so it gets
its own generator. This sim has no real attention scores to rank tokens
by (that ranking is H2O's actual research contribution), so it models the
footprint *consequence* of a token budget rather than the scoring itself:
once past the budget, resident token count is capped instead of growing.

## Golden cross-check

Before trusting the sweep's own bookkeeping, `golden/check_closed_form.py`
independently re-derives the formula from scratch and checks it against
`sim/workload.py`'s output on 500 steps — the same role a golden model
plays checking RTL on the hardware-design side of this portfolio:

```
PASS: sim/workload.py matches the closed-form formula on all 500 steps
```

## Running all four policies against the same 40MB budget

`sim/run_sweep.py` runs baseline, GQA (32 heads → 4 KV heads), KVQuant
(4-bit), and H2O (64-token budget) over 1000 decode steps against a
Llama-2-7B-shaped config (32 layers, 32 heads, head_dim 128, fp16) and a
40MB SRAM budget — matching last week's back-of-envelope 7B-scale number
exactly:

| Policy | Crosses into HBM at step |
|---|---|
| Baseline | 80 |
| KVQuant (4-bit) | 320 |
| GQA (32→4 KV heads) | 640 |
| H2O (64-token budget) | never — stays in SRAM |

<figure>
  <img src="/assets/images/kv-cache-transformers/plot-cache-growth.png" alt="Line chart of live KV-cache size in MB vs decode step for four policies against a dashed 40MB SRAM capacity line — baseline (blue) grows steepest and crosses the line earliest, KVQuant (green) and GQA (orange) grow more slowly and cross later, and H2O (red) flattens out below the line and never crosses">
  <figcaption>KV-cache growth vs. the 40MB SRAM ceiling, all four policies on the same sweep. Baseline crosses almost immediately; H2O never does, because its token budget caps growth below the ceiling entirely.</figcaption>
</figure>

<figure>
  <img src="/assets/images/kv-cache-transformers/plot-bandwidth.png" alt="Line chart of achieved bandwidth in GB/s vs decode step for the same four policies, each holding flat at a high SRAM bandwidth value then dropping sharply to a lower HBM bandwidth value at its own cliff step — H2O's line never drops">
  <figcaption>Achieved bandwidth per policy — a step function, not a gradual decline, because the memory model treats SRAM/HBM as a binary fit/no-fit per step rather than a partial cache.</figcaption>
</figure>

The bandwidth plot is the more direct "why does this matter" picture: it's
not that performance degrades gradually as the cache grows — it falls off
a literal cliff, all at once, the step the cache stops fitting on-chip.
Everything to the left of that step runs at full on-chip bandwidth;
everything to the right runs at HBM bandwidth for the rest of the
generation, for every single subsequent token, because a full-attention
decode step never stops re-reading the whole cache.

## What this sweep is and isn't saying

Worth stating plainly, the same way last week separated capacity from
timing: the SRAM/HBM bandwidth numbers here (19,000 GB/s / 3,350 GB/s) are
illustrative knobs loosely anchored to H100-class hardware, not measured
values — the point of this sweep is the *shape* of the crossover per
policy and how far each technique pushes it back, not a calibrated
absolute throughput number. That calibration is what the still-pending GPU
re-run is for.

## Next

- Sweep across SRAM capacities and model scales instead of one fixed 40MB/
  7B-class point, to see how the cliff itself moves, not just where it
  lands for one config.
- Give H2O a real synthetic attention-score model for *which* tokens get
  evicted, instead of a flat token-count budget.
- The GPU re-run, once TACC or similar access is sorted — real bandwidth
  and timing numbers to calibrate the illustrative ones used here.
