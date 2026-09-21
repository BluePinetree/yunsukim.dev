---
title: "Agent Harness Anatomy"
description: "What an autonomous ML-research pipeline looked like once I took it apart, and what its own numbers did not survive."
fromDate: "2026-08"
toDate: "2026-09"
code: "https://github.com/BluePinetree/agent-harness-anatomy"
types:
  - "research"
  - "open-source"
skills:
  - "Python"
  - "CrewAI"
  - "Reproducibility"
  - "Evidence Auditing"
selected: true
---

## What problem was I trying to solve?

An autonomous research pipeline produces artifacts that look exactly like scientific
output: a result file with numbers in it, and a paper with a methods section and a
conclusion. Neither artifact carries any marking that separates a measurement from a
claim. If the code that wrote the number was itself written by a language model, then the
number is the model's report of the measurement, not the measurement.

## What did I build?

Not a system — a record. This repository is the retrospective teardown of `MARS`, an
archived pipeline I had built and run. One claim per document: six findings, each with the
evidence path behind it and a stated confidence level; fourteen architecture decision
records; a pre-registration file; and an index of the archived runs.

## What actually happened?

Two things, both unflattering.

First, the study's independent variable did not exist in the code. Every framework
invocation in the completed implementation constructed a crew of one agent running one
task, and the largest phase never imported the framework at all. The comparison the project
was built to make was not unfinished — it was structurally untestable. The claim was
withdrawn.

Second, when I later re-derived the figures this repository had published, several of them
did not reproduce. The result-file count and the directory split reproduced exactly. The
headline directory count turned out to depend on a definition that was never stated. The
run-level measurement table could not be reproduced at all. The counting scripts behind the
original figures are in neither repository, so the differences cannot be adjudicated — only
disclosed. The original numbers are left in place, with the recount beside them.

## What did I learn?

The dangerous failures are the ones that report success. A grid search declared over more
than a thousand model fits that never executed, and was recorded as fully verified. A goal
check that passed below its own goal after a unit conversion applied to one side only. A
guard that read the wrong key and fired zero times across the entire archive. `exit code 0`
is not a success signal.

And a figure that cannot be recomputed is testimony — including when the figure is mine.

## What can be verified?

Reproduced on recount: 96 result files; a 122 / 91 split across two directory generations;
44 run directories carrying a non-empty metrics block; 306 distinct metric key names across
those 44, where the most frequent real metric name occurs three times; 23 of the 44
self-reporting success. Disclosed as not reproducible: the originally published directory
count, the run-level measurement table, and the paper-to-failure ratio.

The archived runs were driven by OpenAI models. No Claude model appears anywhere in the
archive, and nothing here claims otherwise.

## Code and evidence

- [Repository](https://github.com/BluePinetree/agent-harness-anatomy)
- [`findings/`](https://github.com/BluePinetree/agent-harness-anatomy/tree/main/findings) — six claims, each with its evidence path and confidence level
- [`evidence/INDEX.md`](https://github.com/BluePinetree/agent-harness-anatomy/blob/main/evidence/INDEX.md) — the archive inventory and the recount note
- [`protocol/preregistration.yaml`](https://github.com/BluePinetree/agent-harness-anatomy/blob/main/protocol/preregistration.yaml)
- [`decisions/`](https://github.com/BluePinetree/agent-harness-anatomy/tree/main/decisions) — fourteen ADRs
- Original source: [`MARS`](https://github.com/BluePinetree/MARS)
