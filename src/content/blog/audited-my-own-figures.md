---
title: "I Audited My Own Released Figures. Some of Them Don't Reproduce."
description: "I spent months documenting how an autonomous research pipeline reported success it had not earned. Then I pointed the same check at my own write-up."
createdAt: 2026-09-20
draft: true
stage: "budding"
authors:
  - yunsukim
tags:
  - reproducibility
  - agentic-ai
  - research-methods
---

On 2026-09-09 I re-derived the figures I had published in a repository whose entire
argument is that numbers produced by language-model-authored code cannot be trusted
without an independent check.

Several of them did not reproduce.

| Originally reported | On recount |
|---|---|
| 96 result files | **96** — reproduced exactly |
| 122 `run_*` and 91 `v3_*` directories | **122** and **91** — reproduced exactly |
| 271 run directories | **Depends on a definition I never stated.** `outputs/` holds 271 entries, of which 270 are directories — the extra entry is a 1,013-byte crash dump. 224 of those directories contain any file. Across `outputs/` and `runs/`, 276 ids have content |
| A 29-run measurement table | **Not reproducible.** 44 directories carry a non-empty metrics block, and no filter I tried yields 29 |
| 16 of 16 failed experiments produced a paper | **Not reproducible as a ratio.** 37 directories hold a `paper.md`; 18 of those had no successful execution. The direction holds; the enumeration does not |
| 77 result files | **Not reproducible.** 96 `result.json`, 342 `result*.json` |

The scripts that produced the original figures are in neither repository. So the
differences cannot be adjudicated — only disclosed.

I left the original numbers where they were and put the recount beside them. This post is
about why, and about what those numbers were supposed to show in the first place.

## What the numbers were for

I had built an ML-research pipeline: natural-language question in, plan, code, execute,
analyze, paper out, with human approval gates. It ran. It produced papers and metric files.
And then it kept producing results that were wrong in a specific, uncomfortable way — they
came back marked successful.

Three of the cases, out of seven I eventually documented.

**A search that never ran.** A hyperparameter sweep was configured for 810 trials. The
result matched the no-search baseline to sixteen decimal places. The configuration had no
path to reach the experiment: the argument was accepted by the launcher and never
forwarded. The sweep never executed once. The system recorded the run as *fully verified*.
I caught it because two numbers were suspiciously identical. Nothing in the pipeline
noticed.

**A goal check that passed below its goal.** A target of 70% was compared against a value
of `0.6886`, after a unit conversion that happened on one side only. 68.86 was accepted as
reaching 70%, and early stopping fired. A goal check that reports success below the goal is
worse than no goal check, because it consumes the budget that would otherwise have kept the
run going.

**A guard that never fired.** A check existed to warn when the executed scale fell below
the planned scale. It read the value from the wrong key. It fired zero times across the
entire archive. A dormant guard is worse than a missing one: a missing guard is a known
gap, a dormant guard is a false assurance that something is being watched.

They share a shape. Process exit code: 0. Pipeline status: complete. Self-reported outcome:
success. Actual outcome: wrong, and undetectable from the artifacts.

Four of the first five were found by hand, by noticing something odd about a number. None
were found by the system. Two more were still waiting to be found after I had declared the
list complete — one of them a function whose own docstring describes the failure mode that
the non-Windows branch then implements anyway, invisible because every run was on Windows.

That ratio is the finding, more than any individual case.

## What is already known, and what is not mine

This failure mode has a literature, and it is larger than my archive.

Advani measures what they call false success across two agent benchmarks — 9,876 tau2-bench
trajectories and 1,879 AppWorld trajectories — and finds it in 45–48% of failures in
single-control domains and 75.8% of self-assessing coding-agent trajectories with explicit
status claims. The same work shows that LLM judges do not catch it: no configuration across
five judges and five prompt strategies exceeds 0.65 AUROC on one benchmark, or 0.54 on the
other, because the judges lean on confident closing language rather than verified state
changes ([arXiv:2606.09863](https://arxiv.org/abs/2606.09863)).

Ding et al. survey the wider gap. Of 24 runnable autonomous-research systems, 83% release
code, but only 38% release seeds or execution traces, and only 38% report any
novelty-verification method. Among nine closed-loop systems, seven are mechanical re-runs;
none in their corpus demonstrates an externally validated in-loop oracle
([arXiv:2608.05179](https://arxiv.org/abs/2608.05179)).

Gaddipati et al. push it downstream: in their benchmark of six autonomous research systems,
59% of the submissions accepted by automated review contained fabricated or unsupported
claims ([arXiv:2605.16616](https://arxiv.org/abs/2605.16616)). Zhu et al. argue from 28
generated papers that the bottleneck is not creativity but the capability to execute
verification procedures ([arXiv:2506.01372](https://arxiv.org/abs/2506.01372)).

The terms are not mine. *Silent failure*, *false success*, *confident closing*, *the
verification gap* — all of that is prior work, at a scale I cannot approach. One operator,
one machine, one model family, conditions changing as the system was debugged. What I have
is not a rate. It is a set of root causes, each with the check that would have caught it.

## Evidence and testimony

The distinction that survived the project is this one:

| | Written by | Admissible as |
|---|---|---|
| **Testimony** | Code the language model authored | Record it. Never adjudicate on it |
| **Evidence** | Code the model is forbidden to rewrite | The only basis for a verdict |

It sounds pedantic until you watch it cost you four months.

Here is why a better parser cannot substitute. Across the 44 archived runs whose result
file carries a non-empty metrics block, there are 306 distinct metric key names. The most
frequent real metric name occurs three times. The only keys that recur more often are
wrapper keys produced by a self-nesting bug. Some metrics blocks contain `error`,
`traceback`, `host`, `cwd`. Any normalisation layer over this is either a hardcoded alias
table that breaks on the next run, or fuzzy matching that puts judgement back inside what
is supposed to be a deterministic check.

Two more properties of that same set of 44 sharpen it. Twenty-three self-report success.
Exactly one carries a metric key naming a validation split — and that one belongs to a
development era the archive's own README marks as uncitable. The protocol required
selecting on validation and evaluating on test once. For practical purposes no run recorded
a validation metric at all, which means the selection rule was unobservable for the entire
life of the project.

The structural fix is cheap, and it is not a parser. A fixed layer owns a separate block in
the result file and writes back what it actually received:

```
result.json
├── metrics          ← generated code. testimony. never adjudicated on
└── verification     ← fixed code only. evidence
    ├── config         the parsed arguments and the actual command line
    ├── splits         a content hash per data partition
    ├── access         how many times the test set was evaluated
    ├── epochs         how many epochs actually ran
    └── selection      what the final model was chosen on
```

Injecting a setting and honouring it are different events. Only the second one matters, and
only the second one is hard to observe.

### The part that was only ever asserted

"The model cannot rewrite the fixed layer" is a security claim, and this project never had
a threat model for it. In practice the fixed files existed as 845 lines of string literals
inside a generator function — not as files. They could not be linted, tested, diffed, or
hashed. The one layer the entire verification argument rested on was the one layer nobody
could inspect.

It was a claim, not a control. Which brings this back to the top of the post.

## Why the original numbers are still up

A figure that cannot be recomputed is testimony. That is the thesis of the work. When I
applied it to my own record, the thesis did not make an exception for me.

I could have quietly replaced the numbers. The repository has few readers; nobody had
challenged them. But silently correcting a figure destroys the only thing that makes the
correction meaningful — the ability of a later reader to see that it happened, and to judge
how large the gap was. A corrected number with no visible history is a number you are again
being asked to take on trust.

So the recount sits at the top of the README, in the evidence index, and in the case study,
with the original figures left where they were. The disclosure is the artifact.

The practical lesson is duller than the epistemics: **keep the counting script.** Every
unreproducible figure in that table failed for the same mundane reason — the code that
produced it was ad hoc and was never committed. A number in a study record is not a result
until the thing that computed it is in version control next to it.

## What this changed about how I work

What I now bring an assistant into, because none of it produces the numbers a verdict rests
on: design critique — asking what would make a planned comparison unanswerable *before*
running it, which is the check that would have ended my three-framework design in an
afternoon; code navigation and seam-reading, since one of the seven defects lives in the
seam between two individually correct functions and another in a branch no run ever
exercised; hypothesis generation about failure mechanisms, treated as something to confirm
in code; documentation; and failure analysis over logs I retain the ability to re-read.

What stays outside, as independently verified: metric truth, dataset splits and their
integrity, trial and epoch counts, stopping criteria, and statistical verdicts. These come
from a fixed layer the model may not author, content-hashed so that a modified evidence
layer is a detected rule violation rather than an undetected success.

That line is now written into a pre-registration file rather than left as prose:
`success_determined_by: harness`, with execution success, validation tier, dataset origin
and evaluation scope listed under fields that are recorded but not trusted. Those four are
precisely the ones a run used to declare a full-validation success while training for a
single epoch.

## One exercise

A generated paper in that archive asserts a 3-epoch budget. Its result file shows one
epoch. Working only from the result file, determine how many epochs ran.

The answer is that `avg_epoch_time_s` equals `total_train_time_s`, which is a fact about
arithmetic rather than about trust. It is derivable, but it should have been *readable* — a
four-line read-back block would have made it so.

That gap, between derivable and readable, is most of what I now build.

---

**Scope.** One system, one operator, retrospective, not independently verified. System
effects and operator effects are confounded and cannot be separated from this record. Every
pathology above is an existence proof from an uncontrolled archive; none of them supports a
rate, and none supports a claim about agent systems in general.

**Provenance.** The archived runs were driven by OpenAI `gpt-5.2`. Every model string
recorded anywhere in the archive is an OpenAI one; no Claude model appears in it. Where
an assistant appears in my current practice, the section above says which parts.

The full record, including the evidence map and the recount, is in
[agent-harness-anatomy](https://github.com/BluePinetree/agent-harness-anatomy).
