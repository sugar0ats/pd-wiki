---
layout: default
title: 2. The Flow and Its Inputs
nav_order: 3
has_children: true
permalink: /docs/flow/
---

# The Flow and Its Inputs

Physical design does not start from nothing. It starts from a netlist someone else
produced, a set of constraints someone else wrote, and a pile of library and technology
files the foundry and library vendor supplied.

This part covers two things: **where physical design sits** in the larger chip design
flow, and **what lands on your desk** when a block is handed to you.

## Pages in this part

1. **[The ASIC design flow]({{ site.baseurl }}/docs/flow/asic-design-flow.html)** — RTL to
   GDSII end to end, and where the place-and-route steps fit.
2. **[Design inputs and file formats]({{ site.baseurl }}/docs/flow/design-inputs.html)** —
   netlist, SDC, LEF, Liberty, DEF, SPEF, SDF, CPF, and what each one is for.

{: .tip }
> A surprising fraction of "the tool is doing something insane" turns out to be a missing
> or wrong input file. Before debugging the tool, check that every collateral was read in
> cleanly — `checkDesign -netlist` before floorplanning exists for exactly this reason.
