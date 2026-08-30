---
title: "Week 4: The autonomous bottleneck report, checked against real literature, and the wrap"
date: 2026-02-09
project: archai
tracks: [ppa-analysis]
summary: "Gemini's own Results and Analysis report names the actual bottleneck, the Deep Research console checks that conclusion against published cache-sizing literature, and — a hackathon submission later — what's next."
---
Closing this arc the same way it started: not by assuming the system's
conclusions are right because they sound plausible, but by checking. This
week is the report ARCHAI wrote about its own experiment, an independent
literature check against that report's central claim, and the actual
submission this was all building toward.

## The report ARCHAI wrote about its own experiment

<figure>
  <img src="/assets/images/archai/archai-bottleneck-report.jpg" alt="ARCHAI dashboard Results and Analysis tab showing a generated report with sections 'Identification of Bottlenecks' and 'Evaluation of Autonomous Decision Making', plus a Recreate Report button">
  <figcaption>The Results and Analysis tab: Gemini's own synthesis of every phase's trial data into a single bottleneck finding.</figcaption>
</figure>

On the two phases from Week 3, verbatim:

> The primary microarchitectural bottleneck identified was the L1
> Instruction Cache Capacity at sizes below 4kB. When the L1I is constrained
> to 1kB or 2kB, the frontend suffers from misses during the execution of
> sorting loops and library calls, leading to pipeline stalls.
>
> Conversely, the L1 Data Cache and DDR Capacity are not bottlenecks for
> this workload. The 100-element array manipulation is small enough to
> reside entirely in the L1D, and the total memory footprint is so minimal
> that DDR capacity scaling provides no benefit.

And on its own process, under "Evaluation of Autonomous Decision Making":

> The ARCHAI system (Gemini 3) demonstrated high-order reasoning in its
> runtime modifications. Upon observing the lack of sensitivity to L1D in
> Phase 0, the agent correctly pivoted the experiment to focus on "hardware
> minimalism."

That's a specific, falsifiable claim — L1I below 4kB is the bottleneck,
nothing else is — not a vague "cache size matters" hand-wave. Specific
enough to actually check against something outside this one stressor.

## Checking the conclusion against real literature

<figure>
  <img src="/assets/images/archai/archai-deep-research-validation.jpg" alt="ARCHAI dashboard Research Console showing a Deep Research cross-validation report citing Hill and Smith's cache performance models and ARM Cortex-A tuning guides, concluding the experimental results agree with published cache sizing heuristics for small working sets"
>
  <figcaption>The Deep Research console: the L1D-capacity finding, checked against published cache-sizing literature — not asserted, verified.</figcaption>
</figure>

This is the piece scoped back in Week 1 and left unused until there was an
actual claim worth checking: a separate Deep Research model, tasked with
comparing the experiment's findings against outside sources. Its report on
the earlier L1D-capacity plateau (the Phase 0 result that motivated pivoting
toward L1I and DDR in the first place):

> The observed IPC plateau beyond approximately 32kB of L1D cache capacity
> aligns closely with established cache working-set theory. Prior studies,
> such as Hill and Smith's performance models and later evaluations in
> embedded processor design, demonstrate that once the entire active data
> footprint fits within L1, further cache expansion yields diminishing
> returns.
>
> For small array sizes (N = 100), the working set—including stack frames
> and temporary buffers—comfortably fits within 16-32kB. This mirrors
> findings from ARM Cortex-A series tuning guides, which recommend modest
> L1D sizing for control-oriented workloads.
>
> Conclusion: The experimental results strongly agree with published cache
> sizing heuristics for small working sets.

That's the same discipline Week 3 was pushing for, just applied by the
system to itself: don't let a plausible-sounding conclusion stand
unexamined, go check it against something independent.

## Corroboration from a run wider than these six screenshots

The project's own write-up (submitted separately, covering phases beyond
what's captured here) reports the same shape of finding holding up more
broadly: "clear diminishing returns in cache capacity once the working set
fit entirely within L1," "associativity proved more impactful than raw cache
size" for reducing conflict misses, and core-count scaling showing "limited
benefit for small, cache-bound workloads." None of that contradicts anything
above — it's the same conclusion, extended to associativity and core count,
which this particular set of screenshots doesn't individually cover.

## The unsexy infrastructure work underneath all of it

Worth restating plainly: none of the phase-hopping in Weeks 3-4 would have
been practical without Week 1's Docker/GHCR fix. Turning a 30-40 minute
gem5 build into a 1-2 minute pull-and-mount is not a finding anyone puts in
a report, but it's the reason running 10-20 trials across multiple phases
was feasible to iterate on at all instead of purely theoretical.

## What's next

Submitted to the **Gemini 3 Hackathon**. The two directions actually named
as next steps: exploring **custom hardware prefetchers**, and extending the
framework to integrate **RTL and Verilog-level designs** — pushing the
autonomous loop down from architectural parameters (cache sizes, core
counts) to actual hardware description. That's a real bridge to the RTL work
already sitting elsewhere on this site — [Foundation Core](/projects/foundation-core/)'s
pipelined RISC-V core and [SysMAC](/projects/sysmac/)'s systolic array both
went through a synthesis-to-PPA flow by hand; the natural next question for
ARCHAI is whether that same hypothesis-driven loop can drive *that* kind of
exploration too, not just gem5 parameter ranges.
