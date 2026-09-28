---
title: "Agent Harness #3 — The Handoff Problem"
description: "Two channels crossed every phase boundary: the handoff the next stage reads, and the event stream I read. I typed both and enforced neither."
createdAt: 2026-09-25
draft: false
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

Every field on both models the language model actually fills has a default, so the empty
object is a valid plan. Run against the archived code:

```python
>>> PlannerResult.model_validate({})
PlannerResult(problem_statement='', research_questions=[], hypotheses=[],
              success_criteria=[], constraints=[], risks=[],
              recommended_profile='generic_script', next_stage_inputs={})
```

No exception, and nothing in the returned object says it came from nothing. The designer
model behaves the same way, except that its defaults are more confident:
`DesignerResultV4.model_validate({})` hands back `files=[]` with
`entry_point='src/main.py'` — a plan that names a file it never asked anyone to write.

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
into `problem_statement`. Feed both of them the same refusal:

```python
>>> msg = "I need more detail about the dataset before I can plan this."

>>> _parse_planner_result(msg).problem_statement
'I need more detail about the dataset before I can plan this.'

>>> _parse_designer_result(msg)
DesignerResultV4(files=[], entry_point='src/main.py', ...)
```

The model asked a question. Phase 2 receives it as the research problem, together with a
build plan containing no files. Both objects are structurally valid, and they travel on.

And the extractor itself has a fourth fallback. If the JSON is truncated, it walks the
bracket stack and closes it. Take a designer response that hit the token limit partway
through its file list:

```python
>>> raw          # the response stops here, mid-list
'{"entry_point": "src/main.py", "files": [{"path": "src/data.py"}, {"path": "src/model.py"}'

>>> extract_json_object(raw)
{'entry_point': 'src/main.py',
 'files': [{'path': 'src/data.py'}, {'path': 'src/model.py'}]}
```

A plan cut off mid-flight comes back as a complete plan. Nothing records that anything was
dropped, and the recovered object is quietly self-contradictory: it declares an entry point
of `src/main.py` while listing two files, neither of which is `src/main.py`. Phase 2 writes
what it was given. Phase 3 runs a file that was never generated, and reports *File not
found* — three phases from the response that was actually truncated.

<svg viewBox="0 0 500 362" role="img" width="100%" style="max-width:500px;height:auto;display:block;margin:1.5rem auto" aria-label="Diagram of the handoff parsing path. A model response passes down through three gates — extract_json_object, which closes the brackets on truncated JSON; the phase parse function, which returns an empty model and a log warning when no JSON is found; and model_validate, which passes an empty object because no field is required — and arrives as a valid payload. A dashed rail on the right marked raise or halt receives no edge from any gate. Below it sits extract_and_validate, the one helper that returns None on a validation error, which no phase calls.">
<title>Four fallbacks, and the one layer that fails closed</title>
<g fill="currentColor" font-size="10.5" opacity="0.6">
<text x="0" y="12">what a bad response passes through</text>
</g>
<g fill="currentColor" font-size="11" opacity="0.65" text-anchor="middle">
<text x="118" y="22">model response</text>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.25" opacity="0.85">
<rect x="18" y="44" width="200" height="32" rx="5"/>
<rect x="18" y="112" width="200" height="32" rx="5"/>
<rect x="18" y="180" width="200" height="32" rx="5"/>
<rect x="18" y="252" width="200" height="32" rx="5"/>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.25" opacity="0.6">
<path d="M118 28 L118 44"/><path d="M118 96 L118 112"/>
<path d="M118 164 L118 180"/><path d="M118 232 L118 252"/>
<path d="M112 38 L118 44 L124 38"/><path d="M112 106 L118 112 L124 106"/>
<path d="M112 174 L118 180 L124 174"/><path d="M112 246 L118 252 L124 246"/>
</g>
<g fill="currentColor" font-size="11" text-anchor="middle">
<text x="118" y="64">extract_json_object()</text>
<text x="118" y="132">_parse_designer_result()</text>
<text x="118" y="200">model_validate()</text>
<text x="118" y="272">a valid payload, always</text>
</g>
<g fill="currentColor" font-size="9.5" opacity="0.55" text-anchor="middle">
<text x="118" y="90">truncated → brackets closed</text>
<text x="118" y="158">no JSON → empty model + warning</text>
<text x="118" y="226">no required field → {} passes</text>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.15" stroke-dasharray="4 3" opacity="0.4">
<path d="M218 60 L300 60"/><path d="M218 128 L300 128"/><path d="M218 196 L300 196"/>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.15" opacity="0.45">
<path d="M306 56 L314 64"/><path d="M306 64 L314 56"/>
<path d="M306 124 L314 132"/><path d="M306 132 L314 124"/>
<path d="M306 192 L314 200"/><path d="M306 200 L314 192"/>
</g>
<g fill="none" stroke="currentColor" stroke-width="1" stroke-dasharray="3 4" opacity="0.35">
<path d="M420 44 L420 240"/>
</g>
<g fill="currentColor" font-size="10.5" opacity="0.5" text-anchor="middle">
<text x="420" y="34">raise / halt</text>
</g>
<g fill="currentColor" font-size="9.5" opacity="0.5" text-anchor="middle">
<text x="420" y="258">nothing arrives here</text>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.15" stroke-dasharray="4 3" opacity="0.45">
<rect x="280" y="288" width="210" height="38" rx="5"/>
</g>
<g fill="currentColor" font-size="10" opacity="0.6" text-anchor="middle">
<text x="385" y="304">extract_and_validate()</text>
</g>
<g fill="currentColor" font-size="9" opacity="0.5" text-anchor="middle">
<text x="385" y="318">returns None — no phase calls it</text>
</g>
<g fill="currentColor" font-size="10" opacity="0.6" text-anchor="middle">
<text x="250" y="352">The only layer that fails closed is the one nothing calls.</text>
</g>
</svg>

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
high-frequency streaming events, so the lookup always missed:

```python
>>> normalize_event_type("exec_stdout")            # in EVENT_TYPES, lowercase
'AGENT_MESSAGE'
>>> normalize_event_type("token_budget_snapshot")  # in EVENT_TYPES, lowercase
'AGENT_MESSAGE'
>>> normalize_event_type("PHASE_COMPLETE")         # in EVENT_TYPES, uppercase
'PHASE_COMPLETE'
```

Two characters — `.upper()` — and every lowercase event in the system was reclassified as
a generic `AGENT_MESSAGE` before storage.

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
