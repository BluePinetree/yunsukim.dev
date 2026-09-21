---
layout: ../layouts/CommonLayout.astro
title: About
description: Research trajectory, identifiers, and how this site handles numbers.
breadcrumbs:
  - label: About
    icon: research
---

I am a Ph.D. student in the Department of Information and Communication Engineering at
Changwon National University, in SPAI-Lab.

## How the work connects

The publications and the repositories on this site look like two different research
programmes. They are one, and the through-line is measurement.

I came up through signal processing, where the object of study is a physical process you
can only reach through an instrument. Active sonar target classification, then generative
synthesis of sonar signals, then machine fault diagnosis from vibration, then
horizon-adaptive forecasting for coastal metocean data. Different domains, one habit: the
reading an instrument produces is not the quantity being measured, and the gap between
them is where the work is.

Agentic systems break that habit by default. When a language model writes the code that
computes a number, the number it reports is that code's testimony about a measurement, not
the measurement. The pipeline downstream has no way to tell the difference, and neither
does the person reading the paper it writes.

So the current work is instrumentation: building the fixed layer that separates what a
system reports from what can be independently checked, and documenting — stage by stage —
what happens in a research pipeline when that layer is missing.

`physical signals → generative modelling → time-series prediction → autonomous research
systems → verification instrumentation`

## How this site handles numbers

Three rules, because the subject matter demands them.

**Figures that cannot be recomputed are marked as such, not removed.** Several numbers I
published in an earlier write-up did not survive a later recount. The originals are still
in the repository with the recount beside them, because silently correcting a figure
destroys the only thing that makes a correction meaningful — a later reader's ability to
see that it happened.

**Project pages state what can be verified, including when the answer is "not yet."** If a
tool has not been carried through a real research cycle, its page says so. An empty
verification line is information.

**Claims are scoped to the evidence.** One system and one operator is an existence proof,
not a rate. Where a number describes my own archive rather than agent systems in general,
the page says which.

## Identifiers

Name disambiguation matters here — there is at least one other researcher publishing in
AI under the same name, working on machine translation and NLP. These are mine:

- ORCID: [0009-0005-4919-0436](https://orcid.org/0009-0005-4919-0436)
- Google Scholar: [PfZc1IkAAAAJ](https://scholar.google.com/citations?user=PfZc1IkAAAAJ)
- OpenAlex: [A5032267446](https://openalex.org/A5032267446)
- GitHub: [BluePinetree](https://github.com/BluePinetree)
- LinkedIn: [yunsu-kim-2a2b78155](https://www.linkedin.com/in/yunsu-kim-2a2b78155)

## About this site

Built with [Astro](https://astro.build) on the
[myscholar](https://github.com/mychiffonn/myscholar) theme. The source is
[on GitHub](https://github.com/BluePinetree/yunsukim.dev), including the content.
