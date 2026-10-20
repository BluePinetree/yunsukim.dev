---
title: "Agent Harness #5 — Where the Human Goes"
description: "The pipeline stopped to ask a person at five gates. The gates worked; the log could not say who answered. Every verified run's plan was approved by a script in under two seconds."
createdAt: 2026-10-20
draft: false
stage: "budding"
order: 5
authors:
  - yunsukim
tags:
  - agent-harness
  - agentic-ai
  - research-methods
---

> *Agent Harness*, part 5. [Series index](/blog/tags/agent-harness)

In part 3 I wrote that the plan-approval gate "caught things", and said a later post would
weigh how much load it was carrying. This is that post. I went back to the event logs and
counted.

The short version: the gates were built well, they were used far less than the diagram
suggests, and the record cannot tell a person from a script — while the line that tells a
person from a clock is one nothing ever reads.

## The decision

A research pipeline cannot judge its own plan — that was the first thing part 1 said about
research — so somewhere a person has to look at it. That raises two questions: where in a
six-hour run does a person need to be, and what can they do once they are there?

Underneath sat a plumbing problem. The pipeline runs on its own thread; the web interface
the person uses runs on an async event loop. Something has to pause the first and let the
second wake it up again.

## What I did, and what I was reading

ADR-005, dated 2026-05-15, solved the plumbing. Each gate is a `threading.Event`: the
pipeline thread waits on it, and the web endpoint sets it when the person clicks. There is
no polling, and every gate has a timeout, so a run can never hang forever. There are five:

| Gate | Phase | Waits | If nobody answers |
|---|---|---|---|
| Preflight questions | 0 | 60 s per question | uses the default the model wrote alongside its own question |
| Plan approval | 1 | 1 hour | approves the plan |
| Coding guidance | 2 | 2 hours | carries on with automatic repair |
| Execution guidance | 3 | 2 hours | ends the execution phase as failed |
| Context injection | 3 | — | nothing waits; a person may add notes |

ADR-012, dated 2026-06-03, fixed the plan gate's branches. The person has three buttons —
approve, modify, reject — and the original code tested only for approve. Everything else
went back to the planner, so "reject" was read as "modify, with no comment", and the run
asked again. The fix gave reject its own branch: stop the run.

I was not reading anything on human oversight at the time. Both decisions came out of the
bug in front of me and Python's `threading` documentation.

## Does it hold up?

The plumbing holds up. Blocking a thread on an event, releasing it from the web layer and
putting a ceiling on every wait is how I would build it again.

The branching is right, and incomplete. ADR-012 counted three intents, but the plan gate has
four ways out, and the code shows where each one goes:

```python
resolved = gate.wait(timeout=APPROVAL_TIMEOUT_SECS)  # one hour
if not resolved:
    emit("AGENT_MESSAGE", "[Phase 1] Approval timeout — auto-approving plan ...",
         {"auto_approved": True, ...})
    return bundle
if gate.is_approved:
    emit("AGENT_MESSAGE", "[Phase 1] Plan approved by user.", ...)
    return bundle
```

<svg viewBox="0 0 520 252" role="img" width="100%" style="max-width:520px;height:auto;display:block;margin:1.5rem auto" aria-label="Diagram of the four ways out of the plan-approval gate. Approve, no answer for an hour, and asking for changes five rounds in a row all lead to Phase 2 running the plan. Only reject stops the run. Approve and reject are drawn as decisions; the hour of silence and the exhausted modify rounds are drawn dashed, as defaults.">
<title>The four ways out of the plan-approval gate</title>
<g fill="none" stroke="currentColor" stroke-width="1.4" opacity="0.9">
<rect x="0" y="96" width="132" height="104" rx="5"/>
<rect x="372" y="114" width="148" height="44" rx="5"/>
<rect x="372" y="176" width="148" height="34" rx="5"/>
</g>
<g fill="currentColor" font-size="12" text-anchor="middle">
<text x="66" y="142">plan gate</text>
<text x="446" y="140">Phase 2 runs the plan</text>
<text x="446" y="198">run stops</text>
</g>
<g fill="currentColor" font-size="10" opacity="0.6" text-anchor="middle">
<text x="66" y="158">waits for a person</text>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.7">
<path d="M132 112 L340 112 L340 136 L372 136"/>
<path d="M132 193 L372 193"/>
<path d="M366 130 L372 136 L366 142"/><path d="M366 187 L372 193 L366 199"/>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.2" stroke-dasharray="4 3" opacity="0.55">
<path d="M132 138 L340 138"/>
<path d="M132 164 L340 164 L340 138"/>
</g>
<g fill="currentColor" font-size="10.5" opacity="0.8">
<text x="142" y="106">approve</text>
<text x="142" y="132">no answer for an hour</text>
<text x="142" y="158">modify, five rounds running</text>
<text x="142" y="187">reject</text>
</g>
<g fill="currentColor" font-size="9.5" opacity="0.5">
<text x="0" y="236">———  a decision</text>
<text x="0" y="250">- - -  a default</text>
</g>
</svg>

Three of the four lead to Phase 2. An explicit reject is the only exit that stops the run;
asking for changes five times in a row also ends in Phase 2, with the last plan the person
declined.

That is a defensible policy for overnight research runs — nobody wants a run that waits all
night for a click. What it is not is a decision anyone wrote down as one. "Never deadlocks"
is an engineering property; "approve when nobody answers" is a research policy, and
ADR-005 recorded the second only as a consequence of the first.

### What the logs say

The archive holds 171 run logs from May to August 2026. The plan gate opened 58 times:

| At the plan gate | Count |
|---|---|
| Logged "Plan approved by user." | 47 |
| — of those, approved within two seconds | 29 |
| Auto-approved after an hour with no answer | 5 |
| Modify | 2 |
| Reject | 2 |
| Run ended while waiting | 2 |

Twenty-nine approvals in under two seconds are not a person reading a plan. Every approval
from 21 July onward is in that group — 28 of them — and the timing matches the hands-off
runner I wrote that month, to produce benchmark runs with nobody at the keyboard:

```python
# harness/anchor_run.py
ag = services.approval_registry.get(run_id)
if ag is not None and ag.action == "pending":
    services.approval_registry.resolve(run_id, "approve")
```

The runner resolves the gate through the same call the web button uses, so the pipeline
logs it the same way: *Plan approved by user.* The five runs I later treated as the
project's independently verified results are all in this group, approved in 1.0 to 1.9
seconds.

The one-hour timeout is visible in the log, but only to someone who knows where to look: it
is a generic `AGENT_MESSAGE` with an `auto_approved` flag in its metadata, and nothing in the
pipeline, the interface or the write-up ever reads that flag.

The other gates tell the same story. Of 211 preflight questions, an answer was typed for
51; 137 defaults were taken within five seconds — the runner, or someone clicking through.
Of 44 requests for help during coding and execution, 23 waited the full two hours with
nobody there.

None of this is wrong in itself. A benchmark run *should* be hands-off; a reproducible
result should not depend on who was awake. What is wrong is that the record erased the
difference. For every run I cite, the log says a person approved the plan, and none did.

Part 3 needs a correction too. Across the whole archive, the modify and reject buttons were
used four times, and three of those were tests of the buttons themselves. The fourth was
me abandoning a run to start a different one. As far as the log can show, the plan gate never
caught a bad plan.

**What I would do differently now** is two things. `resolve()` would take a required `by`
argument — `"person"`, `"runner"`, or `"timeout"` when the gate expires — and that word
would go into the log line and the result file. *Plan approved by runner* is an honest
sentence. And unattended mode would be something a run declares when it starts, not
something it discovers when an hour runs out.

## What changed since

The design I needed was already published. Magentic-UI, from Microsoft Research in July
2025, separates two places a person matters: *co-planning*, where the person edits and
approves the plan before anything runs, and *action guards*, which ask permission before
actions that cannot be undone, at a frequency the user can configure
([arXiv:2507.22358](https://arxiv.org/abs/2507.22358)). My plan gate is the first. The
second is the principle I lacked for deciding where else a person should be: at the points
that cannot be undone, not at every phase boundary.

The harder critique is older, and comes from policy rather than engineering. Green reviewed
41 policies that require human oversight of government algorithms, and found that people
cannot reliably perform the oversight those policies assume — so the requirement ends up
legitimising the system it was meant to check. In his words, such policies "provide a false
sense of security" ([arXiv:2109.05067](https://arxiv.org/abs/2109.05067)). His remedy is
that any proposed human oversight should come with empirical evidence that it works. The
tables above are that evidence for one small system, and it is thin.

## Transferable

**Putting a person in the loop and recording that a person was there are different jobs.**
A gate whose log cannot say who answered it is a box in the architecture diagram, not
oversight.

If your system has an approval step, find every call that resolves it. Each one should leave
a different name in the log.

## References

**What I was reading at the time (2026-05 – 2026-06)**

- Nothing on human oversight or human-in-the-loop design
- Python `threading` documentation, for the gate mechanism

**Found while writing this post**

- Mozannar, Bansal, Tan et al. — *Magentic-UI: Towards Human-in-the-loop Agentic Systems*
  ([arXiv:2507.22358](https://arxiv.org/abs/2507.22358)), and the
  [Microsoft Research blog post](https://www.microsoft.com/en-us/research/blog/magentic-ui-an-experimental-human-centered-web-agent/)
  describing co-planning and action guards
- Green — *The Flaws of Policies Requiring Human Oversight of Government Algorithms*,
  Computer Law & Security Review 45 (2022) ([arXiv:2109.05067](https://arxiv.org/abs/2109.05067))

---

The system described here is archived at
[`MARS`](https://github.com/BluePinetree/MARS); the decision records are
[ADR-005](https://github.com/BluePinetree/agent-harness-anatomy/blob/main/decisions/ADR-005-hitl-gate-architecture.md)
and
[ADR-012](https://github.com/BluePinetree/agent-harness-anatomy/blob/main/decisions/ADR-012-phase1-reject-vs-modify-branching.md).
The counts come from the 171 archived event logs, May–August 2026.
