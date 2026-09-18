---
layout: default
title: 3. Timing
nav_order: 4
has_children: true
permalink: /docs/timing/
---

# Timing

Almost everything a place-and-route tool does is in service of timing. Placement moves
cells to shorten critical paths. CTS balances a clock tree so edges arrive when they
should. Routing is constrained by what the resulting parasitics will do to delay. Signoff
is largely the process of proving the timing is real.

So it is worth understanding timing *before* the implementation steps, even though it
feels like the abstract part.

## Pages in this part

1. **[STA basics]({{ site.baseurl }}/docs/timing/sta-basics.html)** — what static timing
   analysis is, setup and hold, slack, slew, and delay arcs.
2. **[Clocks, skew, and latency]({{ site.baseurl }}/docs/timing/clocks.html)** — ideal versus
   propagated clocks, skew, insertion delay, uncertainty, and useful skew.
3. **[Variation, corners, and analysis modes]({{ site.baseurl }}/docs/timing/variation.html)** —
   OCV and AOCV, PVT corners, MMMC, and PBA versus GBA.

{: .note }
> Timing is checked repeatedly through the flow against different assumptions. A timing
> report is only meaningful if you know **which stage** produced it and **which view** it
> was run on. See
> [the clock is ideal until it isn't]({{ site.baseurl }}/docs/flow/asic-design-flow/#the-clock-is-ideal-until-it-isnt).
