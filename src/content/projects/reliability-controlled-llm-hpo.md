---
title: "Reliability-Controlled LLM-Guided HPO"
description: "Constraining the selector rather than strengthening the proposer, for low-budget hyperparameter optimization in mixed search spaces."
fromDate: "2026-05"
code: "https://github.com/BluePinetree/reliability-controlled-llm-hpo"
types:
  - "research"
  - "open-source"
skills:
  - "Python"
  - "PyTorch"
  - "Hyperparameter Optimization"
  - "LLM"
  - "Docker"
selected: true
---

## What problem was I trying to solve?

Low-budget hyperparameter optimization in mixed search spaces is brittle. Categorical
choices, noisy early measurements and sparse trial budgets make it easy for a proposal
mechanism to overreact to a candidate that is not actually reliable.

## What did I build?

A reliability-controlled selection layer. The language model is used as a semantic proposal
engine only; what constrains the final choice is downstream of it — candidate validation,
calibration-aware scoring, reliability-coupled risk control, and a reliability-gated
episodic memory. The release is scoped to the manuscript: configs, scripts, figures, and
minimal aggregate artifacts.

## What actually happened?

Two experiments on CIFAR-100 carry the main evidence. A third is included as a failure-mode
stress test, and the repository states plainly that it is a diagnostic rather than
cross-domain proof. The manuscript is under review; citation metadata will be updated when
a public preprint or final version exists.

## What did I learn?

At a low budget, constraining the selector buys more than strengthening the proposer. And
scoping a release to exactly the claims a manuscript makes — then naming that boundary in
the repository — is what keeps a reproducibility claim defensible.

## What can be verified?

Python 3.10.19, CUDA 11.8, PyTorch 2.4.1+cu118, with a Dockerfile and compose target and a
recorded runtime report. Released results include aggregate metrics, seed-level convergence
CSVs, checksummed manifests and the manuscript figure PDFs; an integrity script validates
the delivery against the manifest. There is no paper link yet, because there is not yet a
public one.

## Code

- [Repository](https://github.com/BluePinetree/reliability-controlled-llm-hpo)
- [`results/`](https://github.com/BluePinetree/reliability-controlled-llm-hpo/tree/main/results) and [`figures/`](https://github.com/BluePinetree/reliability-controlled-llm-hpo/tree/main/figures)
- [`CITATION.cff`](https://github.com/BluePinetree/reliability-controlled-llm-hpo/blob/main/CITATION.cff)
