---
title: "MARS — Multi-Agent Research System"
description: "A research question to a structured paper, end to end, with human approval gates. Archived, and the README says why."
fromDate: "2026-06"
toDate: "2026-08"
code: "https://github.com/BluePinetree/MARS"
types:
  - "research"
  - "open-source"
skills:
  - "Python"
  - "CrewAI"
  - "AutoGen"
  - "LangGraph"
  - "React"
  - "FastAPI"
selected: false
---

## What problem was I trying to solve?

Whether an end-to-end ML research pipeline — natural-language question, plan, design, code,
execute, analyze, write — could run on its own with a human intervening at the points that
actually matter.

## What did I build?

A complete CrewAI implementation of that pipeline, a React streaming UI against it, five
human-approval gates, and fourteen architecture decision records. A pre-registered
benchmark protocol was written and committed. Ports to AutoGen and LangGraph were started.

## What actually happened?

The pipeline ran end to end and produced papers and metric files from a natural-language
topic. The benchmark protocol was never executed. The AutoGen and LangGraph ports never ran
end to end. No cross-framework comparison was ever performed.

Earlier versions of this repository opened by calling MARS the first open-source system to
implement and compare the same ML research pipeline across three frameworks. **That claim
is withdrawn.** Every framework invocation in the completed implementation constructed a
crew of one agent running one task, and the largest phase never imported the framework at
all — so the independent variable the comparison depended on was not instantiated in code.

The repository was archived read-only in August 2026. The teardown that followed is
[Agent Harness Anatomy](/projects/agent-harness-anatomy).

## What did I learn?

Without a step that checks — by counting the code — whether a study's independent variable
is actually instantiated, it is possible to spend months on research that is not merely
unfinished but structurally untestable.

## What can be verified?

`ARCHIVED.md` records the archive date, final version, final commit hash and commit count.
The README carries a component-by-component table of what was complete and what was not.
The pre-registration is in the repository, unexecuted. The three prototype directories and
their comparison scaffolding are present and countable.

## Code

- [Repository](https://github.com/BluePinetree/MARS) (read-only)
- [`ARCHIVED.md`](https://github.com/BluePinetree/MARS/blob/main/ARCHIVED.md)
- Successor: [`agent-harness-anatomy`](https://github.com/BluePinetree/agent-harness-anatomy)
