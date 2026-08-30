---
title: "ArchAI"
slug: archai
index: "04"
date_range: "Jan 2–Feb 9, 2026"
icon: 13-network-nodes
subtitle: "AI-Driven Architecture Explorer"
summary: "Autonomous microarchitecture exploration — Gemini 3 orchestrates real gem5 trials, phase by phase, off its own hypotheses."
description: "Gemini 3 orchestrates hypothesis-driven gem5 simulation trials — proposing a phase, running it, reading the real simulator output, and deciding the next phase — checked against published cache literature via a Deep Research pass. Built for the Gemini 3 Hackathon."
tags: [Python, "gem5", Docker, "Gemini 3"]
tracks: [ppa-analysis]
---
Autonomous pre-silicon microarchitecture exploration: Gemini 3 decomposes an
optimization objective into hypothesis-driven experiment phases, runs each
one as real gem5 trials (not a one-shot config or a brute-force sweep),
reads gem5's actual output back in, and decides what the next phase
investigates. A separate Deep Research pass checks the resulting conclusions
against published cache-sizing literature before they're reported. Built
and submitted to the Gemini 3 Hackathon.

Full write-up: four-week blog series starting at
[Week 1](/blog/2026/01/02/archai-week-1-scoping-and-the-gem5-build-wall/).
Source: [GitHub](https://github.com/TanoojMadhuvan/ARCHAI--Autonomous-Research-for-Computer-Hardware-using-AI-).
Submission: [Devpost](https://devpost.com/software/archai).
