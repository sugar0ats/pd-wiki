---
layout: default
title: 1. Fundamentals
nav_order: 2
has_children: true
permalink: /docs/fundamentals/
---

# Fundamentals

Physical design is full of rules that look arbitrary until you know what is physically
happening underneath them. Why does a bigger cell fix a setup violation but cost you
power? Why does a long wire on a low metal layer cause a manufacturing failure? Why does
the tool care so much about capacitance?

All of those answers start at the transistor. This part covers just enough device and
circuit background to make the rest of the wiki make sense — then introduces the
pre-built blocks that physical design actually manipulates.

## Pages in this part

1. **[MOSFETs and CMOS logic]({{ site.baseurl }}/docs/fundamentals/mosfets-and-cmos/)** —
   the two transistor types, how they combine into gates, and where power and delay come
   from.
2. **[Cells: standard, macro, and physical]({{ site.baseurl }}/docs/fundamentals/cells/)** —
   the objects a place-and-route tool moves around.

{: .note }
> If you have taken a digital VLSI course, you can skim this part. The one section worth
> reading carefully is
> [where delay and power come from]({{ site.baseurl }}/docs/fundamentals/mosfets-and-cmos/#where-delay-and-power-actually-come-from),
> because the whole rest of the flow is an argument about those two quantities.
