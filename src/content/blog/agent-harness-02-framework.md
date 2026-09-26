---
title: "Agent Harness #2 — What the Framework Was Actually Doing"
description: "I built the pipeline on a multi-agent framework. Counting the call sites later showed it ran one agent with one task, every time — and that the study had lost its independent variable."
createdAt: 2026-09-22
draft: false
stage: "budding"
order: 2
authors:
  - yunsukim
tags:
  - agent-harness
  - agentic-ai
  - research-methods
---

> *Agent Harness*, part 2. [Series index](/blog/tags/agent-harness)

"I built it on CrewAI" is the kind of sentence you write in a README without examining it.
It was true in the sense that the import was there, in every phase, and the pipeline did
not run without it. It turned out to be false in almost every sense that mattered to the
study I was running.

This post is about how that happened, and about the one architectural decision that made
it happen.

## The decision

Phase 2 of the pipeline generates the experiment: data loaders, models, trainers, an entry
point. Four or five Python files, written by a language model, that Python then has to
execute.

The conventional way to do that in an agent framework is the agent–tool pattern:

<svg viewBox="0 0 420 252" role="img" width="100%" style="max-width:420px;height:auto;display:block;margin:1.5rem auto" aria-label="Vertical diagram of the agent–tool pattern. A FileCoder agent connects by a dashed edge to a WorkspaceWriteTool, annotated as decided by the model, and the tool connects by a solid edge to a file on disk, annotated as always, once called.">
<title>The agent–tool pattern, and which of its two edges is a guarantee</title>
<g fill="none" stroke="currentColor" stroke-width="1.25" opacity="0.85">
<rect x="125" y="20" width="170" height="34" rx="5"/>
<rect x="110" y="96" width="200" height="34" rx="5"/>
<rect x="140" y="172" width="140" height="34" rx="5"/>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.25" stroke-dasharray="4 3" opacity="0.5">
<path d="M210 54 L210 96"/>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.25" opacity="0.6">
<path d="M210 130 L210 172"/>
<path d="M204 90 L210 96 L216 90"/>
<path d="M204 166 L210 172 L216 166"/>
</g>
<g fill="currentColor" font-family="inherit" font-size="12" text-anchor="middle">
<text x="210" y="42">FileCoder agent</text>
<text x="210" y="118">WorkspaceWriteTool</text>
<text x="210" y="194">file on disk</text>
</g>
<g fill="currentColor" font-size="10" opacity="0.6">
<text x="224" y="71">the model decides</text>
<text x="224" y="85">whether this happens</text>
<text x="224" y="155">always, once called</text>
</g>
<g fill="currentColor" font-size="9.5" opacity="0.5">
<text x="0" y="236">- - -  contingent on the model</text>
<text x="0" y="249">———  executed by code</text>
</g>
</svg>

The agent receives the task, decides to call the tool, passes the generated code as the
argument, and the tool persists it. That is the pattern the framework documentation
describes, and it is the pattern I started with.

Only one of those two edges is a guarantee. That turned out to be the whole story.

## What I did, and what I was reading

It did not work, and it did not fail cleanly either.

The model returned Python code **as plain text in the response body** instead of emitting a
tool call. The ReAct loop recorded that text as a Final Answer and moved on. The tool was
never invoked. The workspace stayed empty, and what surfaced downstream was a *File not
found* error several stages later — the first of many failures in this project that
reported themselves as something other than what they were.

My notes from the time record what I tried:

| Mitigation | Outcome |
|---|---|
| Stronger system prompt — "MUST call the tool" | complied about 60% of the time |
| `parallel_tool_calls=False` | no measurable change |
| Raising `max_iter` | more loops, same rate |
| JSON schema in the task description | model sometimes embedded JSON inside prose |
| Prompt restructured as Action / Action Input | about 80%, still non-deterministic |

Eighty percent is a strange number to sit with. It is high enough to look like a prompting
problem and low enough to be unusable. What eventually changed my mind was noticing the
shape of the fix I kept reaching for: every mitigation was an attempt to make the model
*more likely* to call the tool. None of them could make it certain, because whether the
tool fires is decided at inference time by the model, not by my code.

So I removed the agent from the loop:

```python
raw = llm.call([system_msg, user_msg])
content = _strip_fences(raw)        # remove ```python ... ``` wrappers
workspace_path.write_text(content)  # this always runs
```

Two steps. Ask the model for text; write the text to disk. The guarantee is structural —
Python executes the write after the call returns, and it cannot regress without someone
editing that line. The repair loop uses the same shape: broken file plus error message go
into the prompt, corrected code comes back, Python writes it.

I wrote this up as an architecture decision record — a short, dated note giving the
context, the problem, the decision and what it cost, so that a later reader is not left
inferring intent from the code. This one is ADR-001, dated 2026-05-21, and it lists the
alternatives I rejected: forcing
`tool_choice=required` at the API level (the framework did not expose it cleanly across
providers at that version), a pre-flight check with auto-repair (same compliance problem,
one layer down), and a custom wrapper that intercepts the response (fragile, depends on
the response format holding still).

What I did not record, and should have, is that this decision quietly removed the framework
from the largest phase in the system.

## Does it hold up?

The decision itself: yes. I would make it again, and I would make it sooner.

The reasoning that produced it was narrow but sound — a guarantee that depends on a
model's willingness is not a guarantee, and for an operation as cheap as writing a string
to a file, buying certainty with two lines of Python is obviously correct. The trade-offs I
listed at the time were real and I still think they were priced right: I lost the
framework's implicit memory for that phase and replaced it with explicit dependency
injection, which was more precise anyway.

What does not hold up is what I concluded from it, which was nothing. I treated ADR-001 as
a local fix to one phase. It was evidence about the whole system, and I did not read it
that way for months.

Here is what counting would have told me. Every place the pipeline constructs the
framework's execution unit, across all phases, takes the same shape:

```python
Crew(agents=[task.agent], tasks=[task], verbose=False).kickoff()
```

Seven call sites. One agent, one task, every time. No sequential process, no hierarchical
delegation, no agent-to-agent messaging — none of the mechanisms that distinguish a
multi-agent framework from a loop around a model call.

<svg viewBox="0 0 480 248" role="img" width="100%" style="max-width:480px;height:auto;display:block;margin:1.5rem auto" aria-label="Two diagrams side by side. On the left, what a crew can express: a manager delegating to three agents that message each other. On the right, what every call site in the pipeline actually built: one agent and one task, repeated seven times, with no delegation, no messaging and no shared process.">
<title>What the framework could express, and what the pipeline instantiated</title>
<g fill="currentColor" font-size="10.5" opacity="0.6">
<text x="0" y="12">What a crew can express</text>
<text x="264" y="12">What every call site built</text>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.25" opacity="0.85">
<rect x="68" y="28" width="90" height="28" rx="4"/>
<rect x="8" y="104" width="64" height="28" rx="4"/>
<rect x="81" y="104" width="64" height="28" rx="4"/>
<rect x="154" y="104" width="64" height="28" rx="4"/>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.25" opacity="0.5">
<path d="M113 56 L40 104"/><path d="M113 56 L113 104"/><path d="M113 56 L186 104"/>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.15" stroke-dasharray="3 3" opacity="0.45">
<path d="M40 132 C 40 162, 113 162, 113 132"/>
<path d="M113 132 C 113 162, 186 162, 186 132"/>
</g>
<g fill="currentColor" font-size="10.5" text-anchor="middle">
<text x="113" y="46">manager</text>
<text x="40" y="122">agent</text>
<text x="113" y="122">agent</text>
<text x="186" y="122">agent</text>
</g>
<g fill="currentColor" font-size="9.5" opacity="0.55" text-anchor="middle">
<text x="113" y="186">delegation, messaging, process</text>
</g>
<g stroke="currentColor" stroke-width="1" stroke-dasharray="3 4" opacity="0.3">
<path d="M240 20 L240 200"/>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.25" opacity="0.85">
<rect x="272" y="42" width="82" height="28" rx="4"/>
<rect x="382" y="42" width="82" height="28" rx="4"/>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.25" opacity="0.6">
<path d="M354 56 L382 56"/>
</g>
<g fill="currentColor" font-size="10.5" text-anchor="middle">
<text x="313" y="60">1 agent</text>
<text x="423" y="60">1 task</text>
<text x="368" y="94">× 7 call sites</text>
</g>
<g fill="currentColor" font-size="9.5" opacity="0.55" text-anchor="middle">
<text x="368" y="124">no delegation</text>
<text x="368" y="140">no messaging</text>
<text x="368" y="156">no shared process</text>
</g>
<g fill="currentColor" font-size="10" opacity="0.6" text-anchor="middle">
<text x="240" y="228">Nothing on the left is reachable from the code path,</text>
<text x="240" y="243">so all three implementations reduce to the right.</text>
</g>
</svg>

Tool usage was just as lopsided:

| Phase | Tools passed to its agents |
|---|---|
| Planning | `tools=[]` — both agents, none at all |
| Coding | no agents at all — ADR-001 replaced them with direct model calls, so the framework is not imported here |
| Execution | command runner, result reader, file editor, syntax checker — the only real use |
| Writing | one file reader, one report writer |

The coding row is the decision from earlier in this post, seen from the outside. That phase
still calls a language model for every file it generates — what it stopped doing is routing
that call through an agent.

Which leaves the question of what the dependency was buying:

| What the framework provided | Substantial? |
|---|---|
| Role and goal text assembled into a system prompt | No — string formatting |
| **A tool-calling loop** | **Yes — the only real dependency** |
| A model-call wrapper | No — a thin adapter |

One of three.

The coordination everyone would call the interesting part was in my own code: an
orchestrator of roughly 2,350 lines that owned phase sequencing, the repair loop, budget
escalation, the event stream, the cancellation token, and the structured handoffs.

So the system *was* multi-agent — six role-specialised agents with distinct prompts and
structured handoffs between them. It was just not the *framework's* multi-agent system.

And that is the part that was not a local fix. The study this pipeline existed for was a
controlled comparison of three frameworks with everything else held constant. If none of
the distinguishing mechanisms are reachable from the code path, all three implementations
reduce to *calling the model in a fixed order* — the same program with different import
statements. The independent variable was never instantiated.

I found this while estimating how expensive it would be to drop the framework. The estimate
came back cheap, and the reason it came back cheap is the finding.

## What changed since

Two things, in opposite directions.

The frameworks have kept moving. CrewAI now supports structured tool output through Pydantic
models, and the documented patterns for guaranteed invocation have improved — though the
community threads on *forcing* tool output still read very much like my 2026-05 notes, and
the recommended answer is still architectural rather than a flag.

The bigger shift is that the position I arrived at by accident is now a position people
argue for on purpose. Dennis et al. ran the controlled comparison I could not: across three
procedural domains and 200 conversations per condition, putting the whole procedure in the
system prompt and letting the model self-orchestrate beat an external orchestrator using
the same model — 4.53–5.00 versus 4.17–4.84 on a five-point scale, with the orchestrated
system failing on 24% of travel-booking conversations against 11.5% for the in-context
baseline ([arXiv:2604.27891](https://arxiv.org/abs/2604.27891)).

Their framing is worth quoting against my own: external orchestration *may have been
necessary for earlier models*, and frontier model capability has since eroded the case for
it. I had removed the orchestration layer from one phase for reliability reasons, and had
no idea I was sitting on a general result.

## Transferable

**Before attributing a property of your system to a framework, count the call sites and
read what they pass.** A dependency that appears in every file can still be doing one small
thing, and a framework that is present, imported, and load-bearing for exactly one
mechanism will feel foundational right up until you price its removal.

`grep -rn "agents=\[" ` was the whole investigation. It took about a minute, and it was
available to me for months.

## References

**What I was reading at the time (2026-05)**

- CrewAI 1.x documentation on tools and native function calling
- My own failure notes, recorded as ADR-001 on 2026-05-21

**Found while writing this post**

- Dennis, Diamond, Patil, Shabahang, Guo — *In-Context Prompting Obsoletes Agent Orchestration for Procedural Tasks* ([arXiv:2604.27891](https://arxiv.org/abs/2604.27891))
- [CrewAI documentation — Tools](https://docs.crewai.com/en/concepts/tools), on structured tool output via Pydantic models
- [CrewAI community — Forcing Tool Output as Result](https://community.crewai.com/t/forcing-tool-output-as-result/5643)

---

The system described here is archived at
[`MARS`](https://github.com/BluePinetree/MARS); the decision record is
[ADR-001](https://github.com/BluePinetree/agent-harness-anatomy/blob/main/decisions/ADR-001-direct-llm-calls.md)
and the call-site count is
[finding 01](https://github.com/BluePinetree/agent-harness-anatomy/blob/main/findings/01-single-agent-wrapper.md).
