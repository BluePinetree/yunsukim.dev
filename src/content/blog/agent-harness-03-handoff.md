---
title: "Agent Harness #3 — The Handoff Problem"
description: "Two channels crossed every phase boundary: the handoff the next stage reads, and the event stream I read. I typed both and enforced neither."
createdAt: 2026-09-25
draft: true
stage: "budding"
order: 3
authors:
  - yunsukim
tags:
  - agent-harness
  - agentic-ai
  - research-methods
---

> *Agent Harness*, part 3. [Series index](/blog/tags/agent-harness)

The first post ended on the claim that every later failure in this series lives on a
boundary rather than inside a stage. This is the post about what a boundary is made of.

Two things cross every one of them. The **handoff** is what the next phase reads. The
**event stream** is what I read. I designed the first one carefully and the second one by
accident, and the interesting part is that both ended up with the same defect.

## The decision

Phase 1 produces a plan. Phase 2 has to build from it. What travels between them?

The default in almost every agent demo is the transcript: pass the previous agent's full
response forward and let the next one read it. It degrades in a specific way as the
pipeline gets longer. By Phase 4 the context window is mostly Phase 1–3 logs the writing
stage has no use for, and every downstream agent is re-extracting fields out of upstream
prose — which adds a failure mode that has nothing to do with the task (*did the model
correctly read the metric name out of the previous response?*) at every boundary.

I chose the alternative. ADR-002, dated 2026-05-15: all inter-phase handoffs are Pydantic
models serialised as JSON.

## What I did, and what I was reading

`core/handoff_models.py` grew to 26 models. The ones that matter here are `PlannerResult`
and `DesignerResultV4` — the two the language model fills in — and `PlanBundle`, which
Python assembles from them for Phase 2.

Two rules sat on top. The **extraction rule**: each agent is prompted to emit only a JSON
object matching its target schema, `extract_json_object()` pulls it out, and
`model_validate()` checks it. The **persistence rule**: every handoff is written to
`outputs/{run_id}/handoff/` before the next phase starts, so a crash resumes from the last
checkpoint instead of the top of a six-hour run.

The alternatives I rejected at the time: LangChain's `OutputParser`, which still requires
the model to produce parseable text and so moves the problem rather than solving it; a
shared vector store as agent memory, a stateful external service for a pipeline whose phase
order is fixed; plain dataclasses, no coercion; and passing file paths only, fine for logs
and useless for the plan data the approval dialog has to render.

I have the ADR and I have the code. I do not have a reading list. This decision came out of
the Pydantic documentation and the shape of the problem, not out of a literature search —
and for a choice this conventional, that was probably fine.

The ADR's first listed benefit is the one to come back to:

> **Early failure detection.** `model_validate()` raises a validation error the moment a
> required field is absent, at the phase boundary — not three phases later.

## Does it hold up?

The choice does. Typed handoffs over transcripts is still the right call, and the
checkpoint property alone paid for it more than once.

The claim does not. There are no required fields.

```python
class PlannerResult(BaseModel):
    problem_statement: str = ""
    research_questions: list[str] = []
    hypotheses: list[str] = []
    success_criteria: list[str] = []
    constraints: list[str] = []
    recommended_profile: str = "generic_script"
```

Every field on both models the language model actually fills has a default.
`PlannerResult.model_validate({})` returns a valid `PlannerResult`. So does
`DesignerResultV4.model_validate({})` — with `entry_point` set to `"src/main.py"` and
`files` set to `[]`, because those are the defaults. I ran both while writing this post to
be sure. Neither raises.

Three more layers below that, all leaning the same way.

The library helper catches validation errors and returns `None` rather than raising — and
no phase calls it. They all import the raw extractor and do their own handling:

```python
def _parse_designer_result(raw: str) -> DesignerResultV4:
    data = extract_json(raw)
    if not data:
        logger.warning("DesignerResult: no JSON found. raw=%s", raw[:200])
        return DesignerResultV4()
    return DesignerResultV4.model_validate(data)
```

A total parse failure produces an empty designer result and a log line. The planner version
is worse in a more interesting way: it puts the first 500 characters of the raw response
into `problem_statement`, so prose that failed to be JSON becomes the problem statement —
structurally valid, and it travels on.

And the extractor itself has a fourth fallback. If the JSON is truncated, it walks the
bracket stack and closes it. A plan cut off mid-flight comes back as a complete plan, minus
whatever came after the cut, with nothing recording that anything was dropped.

So the boundary was typed, and it was not enforced. Every layer was built to keep going.

That was deliberate, and it deserves a fair hearing: a model that occasionally wraps its
JSON in *"Here is the plan:"* will kill a six-hour run if the parser is allowed to raise,
and I was building something that had to survive its own inputs. What I did not notice is
that I had bought survival by making the contract unable to fail. **A check that cannot
fail is not a check.**

One honest mitigation: Phase 1 output goes to a human for approval before Phase 2 starts,
and an empty file list is visible in that dialog. The gate caught things. But a person
reading the payload is a human doing schema enforcement by eye, and post 5 is about how
much load that gate was quietly carrying.

### The part that is harder to write

Thirteen days later I diagnosed exactly this pattern on the other channel.

Everything the pipeline does is emitted as a typed event, stored, and streamed to the UI —
and every event passes through `normalize_event_type()` first, which uppercased its input
before checking membership. The event type set deliberately held lowercase names for the
high-frequency streaming events, `exec_stdout` and `token_budget_snapshot` among them, so
the lookup always missed and all of them were silently reclassified as generic
`AGENT_MESSAGE`.

Five features sat downstream: the terminal pane, the Phase 2 progress bar, the token budget
warning, the failure escalation alert, the extension proposals sheet. All five were
implemented. None had ever activated. Nothing had errored. The fix, in ADR-007 on
2026-05-28, was to check for an exact match before falling back to case-insensitive.

Then I wrote a section in that ADR called *Engineering Lesson*, naming the pattern — a
normalisation function that silently coerces unrecognised input to a default is a silent
killer for anything gated on the normalised value — and listed three other places in the
codebase where the same thing had happened.

The handoff parser is not on that list. I had written it thirteen days earlier, it is the
same pattern in the same repository, and I did not connect them. **The lesson was recorded
and not applied.** That is a different kind of mistake from not knowing, and it is the one
this project produced most reliably: the finding exists, in writing, and nothing downstream
consumes it.

ADR-006 is the constructive half of the same episode. It replaced the one-size
`AGENT_MESSAGE` with dedicated event types per outcome, and made the renderer registry
distinguish `null` — *stored, deliberately not rendered* — from a missing key,
*unrecognised, fall through*. Two kinds of silence, named separately. That is exactly the
discipline the handoff layer never got.

## What changed since

The parsing problem I was working around is largely solved at the API layer. Provider-level
structured output with strict schema enforcement returns a conforming object by
construction — with two carve-outs: a refusal, and a response that was prematurely
interrupted. The second is precisely my truncation path, and my handling of it was to close
the brackets and carry on. Strict mode would have deleted most of `json_extractor.py` and
left that case exactly where it was.

The more useful correction is that the property I thought I was buying is not the property
structured output provides. Li benchmarks this directly: 2,400 calls to four open models on
a deterministic ordering task, separating syntactic validity from schema validity from
semantic correctness. The strongest model reaches 100% schema validity while semantic
success stays near 80%; in weaker models, schema-valid *unsafe* acceptances run into double
digits. The conclusion is the sentence I needed in May — structured output is a necessary
interface layer, not a substitute for domain verification and fail-closed execution
([arXiv:2607.18261](https://arxiv.org/abs/2607.18261)).

Fail-closed is the property my parser lacked, and no amount of schema strictness would have
given it to me.

Cemri et al. supply the map. Their taxonomy, built from 1,600+ annotated traces across
seven multi-agent frameworks, sorts 14 failure modes into three categories, one of which is
inter-agent misalignment — the boundary itself
([arXiv:2503.13657](https://arxiv.org/abs/2503.13657)). Read against my own repository it
audits where the effort went: I engineered the handoff hard, and the category I left
undefended is the third one, task verification. That is post 7, and everything after it.

## Transferable

**A typed boundary tells you what the payload should contain. It does not tell you the
payload arrived.** Those are different guarantees, and only the second one survives a bad
upstream.

One question separates them, and it can be asked of every contract in a system: *what input
makes this raise?* If the answer is "nothing," the type is documentation.

## References

**What I was reading at the time (2026-05)**

- Pydantic v2 documentation — `model_validate`, field defaults, validators
- CrewAI 1.x documentation on task output and context passing
- No literature search. The alternatives I weighed are recorded in ADR-002 itself; I have
  no other notes from this decision, and inventing a reading list for it would be the kind
  of thing this series exists to catch.

**Found while writing this post**

- Yin Li — *When JSON Is Not Enough: Semantic Reliability of Schema-Constrained LLM Ordering
  Agents* ([arXiv:2607.18261](https://arxiv.org/abs/2607.18261))
- Cemri, Pan, Yang et al. — *Why Do Multi-Agent LLM Systems Fail?*
  ([arXiv:2503.13657](https://arxiv.org/abs/2503.13657))
- [OpenAI — Structured model outputs](https://developers.openai.com/api/docs/guides/structured-outputs),
  on the strict-mode guarantee and its refusal and interruption carve-outs

---

The system described here is archived at
[`MARS`](https://github.com/BluePinetree/MARS); the decision records are
[ADR-002](https://github.com/BluePinetree/agent-harness-anatomy/blob/main/decisions/ADR-002-json-handoff.md),
[ADR-006](https://github.com/BluePinetree/agent-harness-anatomy/blob/main/decisions/ADR-006-event-taxonomy.md)
and
[ADR-007](https://github.com/BluePinetree/agent-harness-anatomy/blob/main/decisions/ADR-007-event-normalization.md).
