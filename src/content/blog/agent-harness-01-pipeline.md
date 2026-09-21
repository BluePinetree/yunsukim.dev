---
title: "Agent Harness #1 — From a Research Question to a Paper: Designing the Pipeline"
description: "I set out to compare three multi-agent frameworks on the same research pipeline. The frameworks moved faster than I could build. This is what the one I finished looked like."
createdAt: 2026-09-21
draft: false
stage: "budding"
order: 1
authors:
  - yunsukim
tags:
  - agent-harness
  - agentic-ai
  - research-methods
---

> *Agent Harness*, part 1. [Series index](/blog/tags/agent-harness)

Systems that take a research question and return a paper already exist. Several of them
have been published, benchmarked, and argued about. Building another one was never the
point.

The question I actually wanted to answer was narrower. By the time I started, three
open-source frameworks had become the default way people wrote multi-agent systems —
CrewAI, AutoGen, LangGraph. Everybody picked one. Nobody seemed to have built the *same*
non-trivial pipeline in all three and reported what the choice actually bought you. That
was the contribution I was after: one long-horizon research pipeline, implemented three
times, compared.

I built the CrewAI one first, because I had to have something working before there was
anything to compare.

It took months, and by the time it ran end to end the ground had moved. All three
frameworks had shipped breaking changes. A comparison of the versions I had started
against would have described a state of the world that no longer existed, and would have
kept decaying while I finished the other two ports. So I stopped.

What I had left was a single, working, unusually well-documented attempt: give it a
research goal and a short config, and it plans an experiment, writes the code, runs it,
analyses the result, and produces a paper — with a human approving at five points along the
way. One person, one system, taken from an idea to something that actually ran.

That record seemed worth keeping, so I took it apart stage by stage. This series is that
teardown: how it was designed, what those design decisions did and did not cover, and — by
the end — what happened when I pointed the system's own verification standard at my own
write-up of it.

This first post is the architecture.

## What automating *research* actually demands

There is a large and growing set of agent systems that complete tasks. Booking a flight,
refactoring a module, filling a form. Research is a different shape, in three ways that
turned out to drive every subsequent decision.

**A task has a right answer; research does not.** You can check a booking against the
itinerary. You cannot check "is this the right experiment" against anything, which means
the system cannot validate its own plan and something else has to.

**A task takes minutes; research takes hours.** A training run is not a request-response
cycle. The harness has to stay alive across it, survive the machine going to sleep, and
still know afterwards what happened.

**A task that fails stops; research that fails produces output anyway.** This is the one I
did not see coming. If a flight booking fails you get an error. If an experiment fails, the
next stage still has a result file to read, still has numbers in it, and still writes the
paper. Failure does not interrupt the pipeline — it propagates through it, wearing the same
clothes as success.

That third property is the seed of everything later in this series.

## Seven stages

The pipeline I settled on:

`question → plan → design → code → execute → analyze → write`

In the implementation these collapse into five phase modules plus an exit gate. What
matters more than the stage list is that each boundary forced a decision I did not
anticipate having to make:

| Stage | The question it forced |
|---|---|
| **plan / design** | If the system cannot judge its own plan, where does the human go, and what can they actually do — reject, or edit? |
| **code** | A model is writing code that will be executed. What may it write, and what must it never be allowed to author? |
| **execute** | A long run needs supervision. How do you tell "still working" from "hung" from "quietly dead"? |
| **analyze** | When a stage fails, do you retry, escalate, or stop? What does the next stage get told? |
| **write** | The paper stage reads the analysis. What stops it from writing up a number that was never measured? |

Each of those is a post in this series. The last one is where it comes apart.

<svg viewBox="0 0 460 556" role="img" width="100%" style="max-width:460px;height:auto;display:block;margin:1.5rem auto" aria-label="Vertical diagram of the pipeline. Five phase modules and an exit gate, with the five human-in-the-loop gates marked beside the phases they belong to, and the boundaries between modules highlighted.">
<title>Phase modules, their boundaries, and the five human gates</title>
<g fill="none" stroke="currentColor" stroke-width="1.25" opacity="0.85">
<rect x="8" y="16" width="240" height="46" rx="5"/>
<rect x="8" y="106" width="240" height="46" rx="5"/>
<rect x="8" y="196" width="240" height="46" rx="5"/>
<rect x="8" y="286" width="240" height="46" rx="5"/>
<rect x="8" y="376" width="240" height="46" rx="5"/>
<rect x="8" y="466" width="240" height="34" rx="5"/>
</g>
<g stroke="currentColor" stroke-width="1.25" opacity="0.5">
<path d="M128 62 L128 106"/><path d="M128 152 L128 196"/><path d="M128 242 L128 286"/>
<path d="M128 332 L128 376"/><path d="M128 422 L128 466"/>
</g>
<g stroke="currentColor" stroke-width="1" stroke-dasharray="3 3" opacity="0.35">
<path d="M8 84 L452 84"/><path d="M8 174 L452 174"/><path d="M8 264 L452 264"/>
<path d="M8 354 L452 354"/><path d="M8 444 L452 444"/>
</g>
<g fill="currentColor" font-family="inherit" font-size="12.5">
<text x="22" y="37">phase0_workspace</text>
<text x="22" y="127">phase1_planning</text>
<text x="22" y="217">phase2_coding</text>
<text x="22" y="307">phase3_execution</text>
<text x="22" y="397">phase4_writing</text>
<text x="22" y="488">exit_gate</text>
</g>
<g fill="currentColor" font-size="10.5" opacity="0.6">
<text x="22" y="52">question intake</text>
<text x="22" y="142">plan · design</text>
<text x="22" y="232">code</text>
<text x="22" y="322">execute · analyze</text>
<text x="22" y="412">write</text>
</g>
<g opacity="0.75">
<g fill="currentColor"><circle cx="262" cy="34" r="3"/><circle cx="262" cy="124" r="3"/><circle cx="262" cy="214" r="3"/><circle cx="262" cy="300" r="3"/><circle cx="262" cy="318" r="3"/></g>
<g stroke="currentColor" stroke-width="1" opacity="0.5">
<path d="M248 34 L259 34"/><path d="M248 124 L259 124"/><path d="M248 214 L259 214"/><path d="M248 300 L259 300"/><path d="M248 318 L259 318"/>
</g>
<g fill="currentColor" font-size="11">
<text x="272" y="38">Preflight clarification</text>
<text x="272" y="128">Plan approval</text>
<text x="272" y="218">Coding guidance</text>
<text x="272" y="304">Execution guidance</text>
<text x="272" y="322">Context injection</text>
</g>
</g>
<g fill="currentColor" font-size="10" opacity="0.55" text-anchor="end">
<text x="452" y="81">boundary</text>
<text x="452" y="171">boundary</text>
<text x="452" y="261">boundary</text>
<text x="452" y="351">boundary</text>
<text x="452" y="441">boundary</text>
</g>
<g fill="currentColor" font-size="10.5" opacity="0.6">
<text x="8" y="528">● the five human gates</text>
<text x="8" y="546">- - -  where state crosses, and where every later failure in this series lives</text>
</g>
</svg>

The five human gates are not evenly spread, and that is the point: two of them sit inside
the execution phase, because that is where a long run can go wrong in ways nobody planned
for. The dashed lines are the boundaries. Everything that broke later broke on one of them.

## What was built, and what was not

The completed implementation is in CrewAI, and it works end to end. Some shape:

| Module | Lines |
|---|---|
| `phase3_execution.py` | 1,342 |
| `phase2_coding.py` | 1,209 |
| `phase4_writing.py` | 657 |
| `phase1_planning.py` | 322 |
| `phase0_workspace.py` | 94 |
| `exit_gate.py` | 131 |

Around 14,600 lines across the CrewAI implementation, a React streaming UI on a FastAPI
backend, five human-approval gates, and fourteen architecture decision records written as I
went.

The honest other half of that table:

| Component | State |
|---|---|
| CrewAI pipeline, phases 0–4 | **Complete.** Ran end to end; produced papers and metric files from a natural-language topic |
| React streaming UI | **Complete** against the CrewAI backend |
| Human-in-the-loop gates (5) | **Complete** |
| Fourteen ADRs | **Complete** |
| Pre-registered benchmark protocol | **Written and committed. Never executed** |
| AutoGen port | **Partial.** Never ran end to end |
| LangGraph port | **Partial.** Never ran end to end |
| Cross-framework benchmark results | **None** |

I am putting that second table in the first post deliberately, because it contains the
thing I got wrong.

Stopping because the frameworks had moved was a judgement call I still think was right.
But when I went back through the code much later, I found two further reasons the
comparison could not have worked even if I had finished it — one in the implementation,
one in the measurement design. Neither was visible to me at the time. Those are the next
post, and the repository has carried the withdrawal since it was archived.

## One thing I found while writing this

Writing this post meant counting lines I had not counted before, and one of them contradicts
my own record. The repository's evidence map says the *largest* phase does not import the
framework, citing `phase2_coding.py` at 1,209 lines. `phase2_coding.py` does not import
`crewai` — that part holds, and it is the striking part. But it is the second-largest phase.
`phase3_execution.py` is 1,342 lines.

Small, and it does not change the finding. I am mentioning it because it is the first thing
this series turned up, on the first pass, in a record I wrote myself and considered
finished. It will not be the last.

## Where this goes

The next few posts are design: what the framework was actually doing, how state crosses the
boundaries between stages, what a model may and may not author, where the human sits, and
what it takes to keep a multi-hour run observable. Then: what "success" even means to a
harness, the failures that report success anyway, and finally the same standard applied
back to my own numbers.

The transferable point of this one is smaller than the architecture diagram suggests: **the
number of stages is not the design. The contract between them is.** Every failure later in
this series lives on a boundary, not inside a stage.

---

The system described here is archived at
[`MARS`](https://github.com/BluePinetree/MARS); the analysis is in
[`agent-harness-anatomy`](https://github.com/BluePinetree/agent-harness-anatomy).
The archived runs were driven by OpenAI `gpt-5.2`.
