---
layout: default
title: 4. Implementation
nav_order: 5
has_children: true
permalink: /docs/implementation/
---

# Implementation

The hands-on part of the flow: taking an initialised design and physically building it. Floorplan, power, placement, clock tree, routing — in that order, because each step depends on the last.

## Pages in this part

1. **[Floorplanning]({{ site.baseurl }}/docs/implementation/floorplanning/)** — Die and core sizing, macro placement, blockages and halos, guides, regions and fences, I/O pin assignment, and row creation.
2. **[Power Planning]({{ site.baseurl }}/docs/implementation/power-planning/)** — Rings, stripes, and rails; the power distribution network; special routing; and why IR drop is a floorplan problem.
3. **[Placement]({{ site.baseurl }}/docs/implementation/placement/)** — Global, incremental, and detail placement; placement optimisation; cell padding; spare cells; scan chains and scan reordering; JTAG.
4. **[Clock Tree Synthesis]({{ site.baseurl }}/docs/implementation/cts/)** — Building and balancing the physical clock tree: skew groups, clock tree specs, CCOpt, H-trees and meshes, useful skew, and NDRs on clock nets.
5. **[Routing]({{ site.baseurl }}/docs/implementation/routing/)** — Global route, track assignment, detail route and repair; congestion analysis; NDRs, shielding, via pillars, and yield optimisation.
