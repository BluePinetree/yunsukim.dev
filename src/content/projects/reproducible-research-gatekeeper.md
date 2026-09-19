---
title: "Reproducible Research Gatekeeper"
description: "An Agent Skill that keeps staged computational research reconstructable — from question, through protocol freeze, to audit and handoff."
fromDate: "2026-09"
code: "https://github.com/BluePinetree/reproducible-research-gatekeeper"
types:
  - "tool"
  - "open-source"
skills:
  - "Python"
  - "Agent Skills"
  - "Claude Code"
  - "Codex"
  - "Reproducibility"
selected: true
---

## What problem was I trying to solve?

In staged computational research, the sequence — question, protocol freeze, execution,
audit, reporting, handoff — comes apart quietly. By the time a later session picks the work
up, what was actually run, what was decided, and on what basis is no longer recoverable
from the artifacts. This is the failure I had already watched cost me a project, documented
in [Agent Harness Anatomy](/projects/agent-harness-anatomy).

## What did I build?

An Agent Skill for Codex and Claude Code. It adds one canonical, machine-readable research
status record; prospective, exploratory and confirmatory study labels; explicit
verification-versus-validation language; four audit levels that stop an automated check
from being written up as independent review; preservation of failed, null, excluded and
inconclusive runs; and plain-language handoffs that carry exact evidence paths and the
conditions for re-entry.

State lives in two files: a canonical status record, and an append-only event log. A
dependency-free helper CLI initializes, freezes, validates and summarizes the workflow.
Two operating profiles differ in how much ceremony they impose, but not in the scientific
invariants — so "I was in a hurry" cannot become "I skipped the gate."

## What actually happened?

The skill is released and its structure is inspectable. It has not yet been carried through
a full research cycle of my own from question to handoff, and this page will say so until
it has been. A gate that has never refused anything has not been observed working.

## What did I learn?

Discipline has to be a gate that can refuse, not a document that can be read. And the
invariants have to be separated from the operating intensity, or the intensity setting
becomes the escape hatch.

## What can be verified?

`SKILL.md` states the invariants and the state schema; both can be read without running
anything. Tests cover the operating profiles, platform handling and gate behaviour. The
repository is also explicit about what it does not do: it does not judge whether a
scientific design is substantively correct, and it does not replace domain review, ethics
review, statistical expertise, or regulatory controls.

## Code

- [Repository](https://github.com/BluePinetree/reproducible-research-gatekeeper)
- [`SKILL.md`](https://github.com/BluePinetree/reproducible-research-gatekeeper/blob/main/SKILL.md)
- [`docs/install-claude.md`](https://github.com/BluePinetree/reproducible-research-gatekeeper/blob/main/docs/install-claude.md)
- [`tests/`](https://github.com/BluePinetree/reproducible-research-gatekeeper/tree/main/tests)
