---
title: "Week 1: An LLM that runs its own gem5 experiments — scoping ARCHAI"
date: 2026-01-02
project: archai
tracks: [ppa-analysis]
summary: "Scoping an autonomous pre-silicon exploration tool — Gemini 3 as the reasoning engine, gem5 as the simulator — and solving the first real blocker before a single trial could run: a 30-40 minute build sitting in the way of every iteration."
---
New project, built for the Gemini 3 Hackathon: **ARCHAI**, an autonomous
pre-silicon microarchitecture exploration tool. The premise is narrower than
it sounds — not "AI designs chips," but something more specific and more
testable: can an LLM actually run a real experimental loop against a real
cycle-level simulator, the way a computer architecture researcher would, and
not just generate a config file and call it done.

## What "autonomous" needs to actually mean

A single prompt that spits out a gem5 config isn't research, it's
autocomplete. The bar for this project was higher: the system needed to
**decompose an optimization objective into a sequence of hypothesis-driven
phases**, each phase makes a specific, falsifiable prediction, gets executed
as real gem5 trials, and the actual results — not an assumption about what
they'd be — determine what the next phase investigates.

That requirement shapes the whole architecture. It's not enough for the
model to produce numbers; it has to be able to read gem5's own output back in
and reason over it. So the design settled on three pieces working together:

- **Gemini 3 Flash** as the reasoning engine, using structured JSON output
  and function calling — not free-text — so its experiment plans and phase
  decisions are machine-parseable, not something a script has to guess at
  from prose.
- **gem5**, running an ARM target, as the actual source of truth — every
  claim the model makes has to be backed by a real simulated trial, not a
  memorized number.
- A **Streamlit dashboard** as the human side of the loop: configuring which
  parameters are in play and their ranges, watching phases run, reading the
  final report.

(A fourth piece — a Deep Research model for cross-checking findings against
published literature — is part of the design from the start too, but doesn't
get exercised until there's an actual finding worth checking. More on that in
Week 4.)

## First wall, before a single trial ran: the gem5 build itself

gem5 isn't a pip-installable tool — it's compiled from source with `scons`,
targeting a specific ISA. Doing that fresh inside Google AI Studio's
execution environment cost **30-40 minutes** before the first simulation
could even start. For a system meant to run 10-20 trials per phase across
multiple phases, a 30-40 minute tax on every fresh session isn't a rough
edge, it's disqualifying.

The fix: stop rebuilding gem5 at all. The Dockerfile builds gem5 **once**,
starting from `ghcr.io/gem5/ubuntu-22.04_all-dependencies:v23-0`, cloning
`gem5.googlesource.com/public/gem5` and compiling the ARM target
(`scons build/ARM/gem5.opt -j$(nproc)`) directly into the image. That image
gets published to GitHub Container Registry
(`ghcr.io/tanoojmadhuvan/archai-gem5:latest`) — a build artifact, not
something rebuilt per session.

From there, the actual ARCHAI code (the orchestrator, the stressor, the
dashboard) gets mounted into the running container as a volume, at
`/gem5/configs/example/gem5_library/archai`, instead of being baked into the
image. That split matters: the *simulator* is the slow, rarely-changing
part, so it gets frozen into a pulled image; the *project code*, which
changes every few minutes during development, never triggers a rebuild at
all. Pulling the image plus mounting the code turned the 30-40 minute wall
into a 1-2 minute one.

## What existed by the end of the week

A scaffolded repo, not yet a working loop:

- `main.py` — where the orchestration logic will live: Gemini interactions,
  phase generation, running trials, parsing results.
- `uarch_spec.py` — the gem5-side configuration, parameterized from JSON
  rather than hardcoded.
- `uarch_stressor.c` — the C workload gem5 will actually execute.
- `presilicon_dashboard.py` — the Streamlit front-end.
- `params.json` / `defaultparams.json` / `loadparams.json` — runtime state,
  a baseline template, and saved-snapshot recovery, respectively.

**Next:** get an actual gem5 config talking to a real workload, and see what
one real trial's output looks like before trying to make Gemini run several
of them autonomously.
