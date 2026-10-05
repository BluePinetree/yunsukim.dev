---
title: "Agent Harness #4 — Letting a Model Write Code You Then Run"
description: "Phase 2 let a model write the experiment code. Deciding which files it may write was the right idea, half-built. The check that guarded execution skipped every file importing numpy."
createdAt: 2026-09-30
draft: false
stage: "budding"
order: 4
authors:
  - yunsukim
tags:
  - agent-harness
  - agentic-ai
  - research-methods
---

> *Agent Harness*, part 4. [Series index](/blog/tags/agent-harness)

The first post listed one question per stage. For the code stage it was this: *a model is
writing code that will be executed — what may it write, and what must it never be allowed
to author?*

This post is how Phase 2 answered it. Most of the answer held up. The part that did not was
not where I expected.

## The decision

Phase 2 turns an approved plan into a workspace of Python files with real dependencies
between them — data loading, features, models, training, evaluation — which Phase 3 then
runs as `python src/main.py`. Three things had to be decided before any of that could work:

- **Order and context.** In what order are the files generated, and what does the model see
  when it writes each one?
- **The import contract.** The files will be run a specific way. What import style does
  that require, and how is it enforced?
- **The pre-flight check.** Something has to catch a broken file before a multi-hour run
  finds it.

Under all three sat a fourth decision that I did not write an ADR for: which files belong
to the model at all.

## What I did, and what I was reading

**Order and context** is ADR-004, dated 2026-05-21. Each file the model writes is assigned
a stage — 1 for config and utilities, 2 for data and models, 3 for the experiment entry —
and generated in dependency order. The important half is what goes into each prompt: not a
description of the files it depends on, but their actual source, already written.

```
Workspace dependencies (already written; your imports must match their actual exports):
# === src/config.py ===
<actual source>
```

The ADR's own example says why this matters. Told that a module exports `get_dataloaders`,
a model will invent a signature for it. Shown
`def get_dataloaders(config: Config, split: str = 'train')`, it calls that.

**Ownership** was never an ADR, but it is the most consequential line in the phase. Some
files in every workspace are not generated at all. They are rendered by the harness from
templates — the command-line interface, the entry point, and the function that writes
`result.json`. The model writes the experiment; the harness owns how it is launched and how
its result is recorded.

<svg viewBox="0 0 500 316" role="img" width="100%" style="max-width:500px;height:auto;display:block;margin:1.5rem auto" aria-label="Diagram of a Phase 2 workspace. On the left, harness-owned files rendered by builder.py: cli.py, metrics.py, main.py and artifacts.py, which writes result.json. On the right, model-written files generated in dependency order: stage 1 config and utilities, stage 2 data, models and training, stage 3 experiment_impl.py, with the actual source of earlier stages injected into each later prompt. Stage 2 imports metrics.py; main.py calls stage 3; stage 3 returns a result to artifacts.py.">
<title>Who owns which file in a Phase 2 workspace</title>
<g fill="currentColor" font-size="11" opacity="0.75">
<text x="0" y="14">harness-owned</text>
<text x="280" y="14">model-written</text>
</g>
<g fill="currentColor" font-size="9.5" opacity="0.5">
<text x="0" y="28">rendered by builder.py, never generated</text>
<text x="280" y="28">generated in dependency order</text>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.4" opacity="0.9">
<rect x="0" y="44" width="200" height="30" rx="5"/>
<rect x="0" y="112" width="200" height="30" rx="5"/>
<rect x="0" y="180" width="200" height="30" rx="5"/>
<rect x="0" y="248" width="200" height="30" rx="5"/>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.15" stroke-dasharray="4 3" opacity="0.7">
<rect x="280" y="44" width="220" height="30" rx="5"/>
<rect x="280" y="112" width="220" height="30" rx="5"/>
<rect x="280" y="180" width="220" height="30" rx="5"/>
</g>
<g fill="currentColor" font-size="11" text-anchor="middle">
<text x="100" y="63">cli.py</text>
<text x="100" y="131">metrics.py</text>
<text x="100" y="199">main.py</text>
<text x="100" y="267">artifacts.py → result.json</text>
<text x="390" y="63">stage 1 · config, utils</text>
<text x="390" y="131">stage 2 · data, models, training</text>
<text x="390" y="199">stage 3 · experiment_impl.py</text>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.2" opacity="0.55">
<path d="M390 74 L390 112"/><path d="M384 106 L390 112 L396 106"/>
<path d="M390 142 L390 180"/><path d="M384 174 L390 180 L396 174"/>
<path d="M200 195 L280 195"/><path d="M274 189 L280 195 L274 201"/>
<path d="M390 210 L390 263 L200 263"/><path d="M206 257 L200 263 L206 269"/>
</g>
<g fill="none" stroke="currentColor" stroke-width="1.1" stroke-dasharray="2 3" opacity="0.5">
<path d="M280 127 L200 127"/><path d="M206 121 L200 127 L206 133"/>
</g>
<g fill="currentColor" font-size="9.5" opacity="0.6">
<text x="398" y="97">source injected</text>
<text x="398" y="165">source injected</text>
<text x="240" y="121" text-anchor="middle">imports</text>
<text x="240" y="189" text-anchor="middle">calls</text>
<text x="300" y="257" text-anchor="middle">returns a result</text>
</g>
<g fill="currentColor" font-size="9.5" opacity="0.5">
<text x="0" y="308">- - -  written by the model        ———  owned by the harness</text>
</g>
</svg>

**The import contract** is ADR-009, dated 2026-06-01. Run as a script, `src/` has no
package context, so a relative import that looks perfectly reasonable fails at launch:

```
$ python src/main.py
ImportError: attempted relative import with no known parent package
```

The decision was to forbid relative imports in every generation and repair prompt, with
concrete examples derived from the files actually written.

**The pre-flight check** imports each generated file once, before Phase 3 spends hours
running anything. ADR-010 records the bug that nearly made it useless. This config file, of
the kind Phase 2 writes, is fine:

```python
from __future__ import annotations
from dataclasses import dataclass

@dataclass
class Config:
    learning_rate: float = 1e-3
```

Yet it failed the check every time:

```python
>>> spec = importlib.util.spec_from_file_location("_chk", "src/config.py")
>>> mod = importlib.util.module_from_spec(spec)
>>> spec.loader.exec_module(mod)
AttributeError: 'NoneType' object has no attribute '__dict__'
```

The check did not use a plain `import`. To load a file from an arbitrary path under a
temporary name, it used the three lower-level calls above — and those skip a step `import`
performs for you: registering the module in `sys.modules`, Python's table of loaded
modules. `@dataclass` needs that table. With `from __future__ import annotations`, the type
`float` is stored as the string `"float"`, and to resolve it the decorator looks up the
class's own module in `sys.modules`. It isn't there yet, so it gets `None`. A correct file
failed, and the repair loop then spent its attempts "fixing" code that wasn't broken. The
fix is the missing step, one line: `sys.modules["_chk"] = mod` before `exec_module`.

What I was reading, for all three, was the traceback. The ADR calls the `sys.modules`
behaviour undocumented. It is not quite: the official `importlib` recipe for importing a
source file directly contains exactly that line. It just never says why.

## Does it hold up?

**Order and context: yes, and it is the best decision in the phase.** Giving the model the
real source of its dependencies turns a guessing problem into a reading problem. The ADR
puts the effect at "from ~40% of runs to near zero". There is no run list or script behind
that number anywhere in the repository, so read it for what it is — an impression written
down on the day. The mechanism does not need it.

**Ownership: the right idea, half-implemented.** `metrics.py` started as a model-written
file. A docstring in the template builder records what happened: the training and
evaluation files import `batch_topk_accuracies`, and when the model wrote `metrics.py` it
kept giving that function a different name. The fix was not a stronger prompt. The harness
now writes a canonical `metrics.py` into the workspace before Phase 2 starts, and the repair
loop is forbidden to touch it.

That is the right move, and it was only made halfway. Which files the harness owns is
written down in four places, and they disagree:

| Where | Lists `metrics.py` as harness-owned? |
|---|---|
| The template builder that renders the files | yes, for two scaffold types |
| The repair loop's exclusion set | yes |
| The workspace manifest | no |
| The Designer prompt — *"Do NOT include stable scaffold files"* | no |

The last row is the one that matters. Phase 2 generates every file the Designer's plan
lists, and writes it without checking whether the harness already put one there. A plan
that includes `src/metrics.py` gets the canonical file overwritten by a model-written one —
the exact failure the move was meant to end. During generation, what kept the model off the
harness's files was a sentence in a prompt, and the sentence did not name this file. I
cannot tell from the archive whether a run ever hit it.

**The import contract: the explanation does not survive a test.** ADR-009 says the Phase 2
check could not have caught relative imports, because `spec_from_file_location` simulates a
package context. It does not. Run against the archived check, a relative import fails with
exactly the error Phase 3 would raise. What the archived check does do is decline to run on
a whole class of files:

```python
_SKIP_IMPORT_PATTERNS = re.compile(
    r"\b(torch|tensorflow|sklearn|cv2|PIL|matplotlib|numpy|pandas|"
    r"scipy|seaborn|plotly|xgboost|lightgbm|catboost|gym|stable_baselines3)\b", ...)

if _SKIP_IMPORT_PATTERNS.search(source):
    return CheckResult(passed=True)
```

Three small files, checked while writing this post:

```python
>>> check_import("src/model_plain.py", ws).passed    # from .config import LR
False
>>> check_import("src/model_numpy.py", ws).passed    # import numpy as np
True                                                 # from .config import LR
>>> check_import("src/model_broken.py", ws).passed   # import numpy as np
True                                                 # from config import does_not_exist
```

One `import numpy` line turns the check off for that file, and a timeout counts as a pass
too. The skip exists for a sensible reason — the test environment did not have `torch`
installed — but it was applied everywhere, including in real runs where the libraries were
there. In an ML pipeline, the files most likely to have an import problem are models,
training and evaluation, which are exactly the files that import these libraries.

I cannot show from the archive which of these let the June bug through: the repository's
history starts on 2026-06-04, three days after the ADR. I can say the explanation I wrote
down is not the mechanism.

**What I would do differently now** is three things. The pre-flight check would run in the
environment Phase 3 uses, with no skip list — if a library is missing, the real run fails
too, so the check should. And the import rule would stop being only a prompt. The phase
already parses every generated file and walks its import statements for other reasons; the
rule it needed was four lines more:

```python
# not in the archive — what I would add now
for node in ast.walk(tree):
    if isinstance(node, ast.ImportFrom) and node.level > 0:
        return CheckResult(passed=False, error_type="import",
                           error=f"relative import at line {node.lineno}")
```

A prompt makes the right code likely. A check makes the wrong code unable to pass. ADR-009
had the first and assumed the second.

The third follows from the ownership table. Which files the harness owns would live in one
list, enforced in code: Phase 2 refuses to write any path on it, whatever the Designer's
plan says.

None of this is isolation, and I should say so plainly: the check and the run both execute
model-written code as my user, on my machine, with no container. For one operator's
research box that was a choice I would still make. For anything shared, it would not be.

One more thing is visible from here, and I will leave it for the last post. The harness side
of the ownership line lives in `scaffolds/builder.py` — 845 lines, of which 424 are the
harness's own files held as twelve string literals and rendered into each workspace. The
line was drawn in the right place. What sits on the harness side of it is not a file anyone
can import, test or diff.

## What changed since

Two results bracket ADR-004.

The first was already published when I wrote it, and I had not read it. CodePlan treats
repository-level editing as planning, deriving the context for each model call from
dependency analysis across the repository — the same idea as injecting the source of
already-written files, at much larger scale. On their benchmark it brought five of six
repositories through validity checks where the baselines brought none
([arXiv:2309.12499](https://arxiv.org/abs/2309.12499)).

The second is a qualification. Li et al. find that models struggle to make full use of
cross-file context even when it is in the prompt, and attribute it to pre-training that
rewards attending to nearby code; targeted training improves exact match by up to 19.7%
([arXiv:2503.15301](https://arxiv.org/abs/2503.15301)). Putting the dependency's source in
front of the model makes the right call possible. It does not make it certain — which is the
whole argument for a check that actually runs.

## Transferable

**When the model keeps breaking a contract, stop telling it harder. Take the file away from
it — in code, in one place.** `metrics.py` did not need a better prompt; it needed a
different owner, and it got one halfway: taken away from the repair loop, left within reach
of the generator.

And for every pre-flight check, read the conditions under which it returns `passed=True`
without running anything. That list is what it does not check.

## References

**What I was reading at the time (2026-05 – 2026-06)**

- Python tracebacks, mostly. No literature search for any of the three decisions.
- Python `importlib` documentation — evidently not the recipe that would have prevented the
  `sys.modules` bug

**Found while writing this post**

- Bairi, Sonwane, Kanade et al. — *CodePlan: Repository-level Coding using LLMs and
  Planning* ([arXiv:2309.12499](https://arxiv.org/abs/2309.12499))
- Li, Zhu, Liu et al. — *aiXcoder-7B-v2: Training LLMs to Fully Utilize the Long Context in
  Repository-level Code Completion* ([arXiv:2503.15301](https://arxiv.org/abs/2503.15301))
- [Python documentation — `importlib`, "Importing a source file directly"](https://docs.python.org/3/library/importlib.html#importing-a-source-file-directly)

---

The system described here is archived at
[`MARS`](https://github.com/BluePinetree/MARS); the decision records are
[ADR-004](https://github.com/BluePinetree/agent-harness-anatomy/blob/main/decisions/ADR-004-staged-code-generation.md),
[ADR-009](https://github.com/BluePinetree/agent-harness-anatomy/blob/main/decisions/ADR-009-generated-code-import-rules.md)
and
[ADR-010](https://github.com/BluePinetree/agent-harness-anatomy/blob/main/decisions/ADR-010-importlib-sys-modules-registration.md).
