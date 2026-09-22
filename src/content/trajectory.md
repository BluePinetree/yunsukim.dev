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
