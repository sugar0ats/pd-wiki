---
layout: default
title: Floorplanning
parent: 4. Implementation
nav_order: 1
---

# Floorplanning
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## Why this step matters more than the others

**Floorplanning** is deciding the physical organisation of the block before any standard
cell is placed: how big the die and core are, where the macros sit, where the I/O pins go,
where the power comes in, and which areas are off limits.

It is the first step of place and route and the one with the longest shadow. Every later
stage inherits its decisions. A congested channel between two badly-placed macros cannot be
fixed by a better router; it can only be fixed by moving the macros, which means redoing
everything downstream. Time spent on the floorplan is the cheapest time in the flow.

![Floorplan view]({{ site.baseurl }}/assets/img/floorplanning-01.png)

A floorplan step covers:

1. Deciding **die and core size**, and therefore target [utilisation]({{ site.baseurl }}/docs/fundamentals/cells/#utilisation).
2. Creating **rows** and defining the core area.
3. Placing **macros**.
4. Assigning **I/O pins**.
5. Adding **blockages and halos**.
6. Defining **guides, regions, and fences** for module placement.
7. **[Power planning]({{ site.baseurl }}/docs/implementation/power-planning/)** — usually
   counted as part of floorplanning, covered on its own page here.

---

## Die, core, and utilisation

The **die** is the whole piece of silicon. The **core** is the area inside it available for
placement, with a margin around the edge for the I/O ring and power rings.

Sizing is driven by target utilisation. Start from the total area of all the cells in the
netlist plus the macros, divide by your target utilisation, and that is roughly the core
area you need. A common starting point is **around 70% core utilisation**, deliberately
leaving room for the buffers, upsized cells, clock tree cells, and ECO fixes that later
stages will insert.

If a design is destined for flip-chip packaging, **bump utilisation** is checked at this
point too — the bump array has its own pitch and placement rules.

### Creating rows

Rows are derived from the **site** definition in the technology LEF, and they tile the core
area. Standard cells are later placed into them.

![Creating rows]({{ site.baseurl }}/assets/img/floorplanning-04.png)

See [sites and rows]({{ site.baseurl }}/docs/fundamentals/cells/#sites-and-rows) for how
rows, sites, and cell heights relate.

---

## Macro placement

Macros — SRAMs, register files, IP blocks — are placed before standard cells and then
locked down. Their positions determine where routing can go, where power can be delivered,
and where standard cells will end up crowded.

![Floorplanning tools in the GUI]({{ site.baseurl }}/assets/img/floorplanning-02.png)

![More floorplan tools]({{ site.baseurl }}/assets/img/floorplanning-03.png)

### Guidelines that usually hold

**Put macros on the periphery, not in the middle.** A macro in the centre of the core
splits the standard cell area into pieces and forces every net that needs to cross the
block to route around it. On the periphery, the macro blocks one edge and leaves a single
contiguous region of core for standard cells.

**Orient macros so their pins face the logic they talk to.** A macro's pins are on
specific edges. Rotating it so the pins point inwards, towards the cells that use them,
saves an enormous amount of routing.

**Group macros that communicate.** Macros connected to each other, or to the same block of
logic, should be adjacent.

**Avoid narrow channels.** The space between two macros is where congestion goes to live.
A channel wide enough for cells but not wide enough for the routing those cells need is
the classic floorplan mistake. Either make the channel genuinely wide, or close it
entirely and put a blockage there.

**Watch the corners.** Routing around macro corners is constrained, and cells placed in
macro corners tend to have poor pin access.

{: .interview }
> *"Why do you place macros on the periphery instead of the centre? What happens to
> routability?"* Central macros fragment the placeable area and force detours, so wire
> lengths rise and congestion concentrates in the channels around them. Peripheral macros
> leave one contiguous core region. The secondary effect is on
> [IR drop]({{ site.baseurl }}/docs/signoff/power-analysis/): a macro in the middle blocks
> power stripes from crossing it, so the region on its far side is further from any supply
> source and droops more.

### Relative floorplans

`create_relative_floorplan` locks components into positions relative to each other, so a
group of macros moves as a unit. Useful when a set of macros has a known good relative
arrangement you want to preserve while you experiment with where the group sits.

### Placement status

Every instance carries a **placement status**, and it matters more than it sounds.

![Instance placement status]({{ site.baseurl }}/assets/img/floorplanning-08.png)

| Status | Meaning |
|:--|:--|
| **Unplaced** | No location yet |
| **Placed** | Has a location, but downstream tools may move it |
| **Fixed** | Has a location and downstream tools may not move it |
| **Cover** | Fixed, and cannot be moved even by ECO |

{: .warning }
> Once you are happy with a macro's position, **set its status to fixed**. Otherwise a
> later placement or optimisation step is free to shuffle it, and you will spend an
> afternoon working out why your carefully-planned floorplan dissolved.

### Concurrent macro and standard cell placement

The usual order is macros first, then standard cells. Sometimes it is better to let the
tool place both at once, so that macro positions are informed by where the logic actually
wants to be rather than by a human's guess.

![Concurrent macro placement commands]({{ site.baseurl }}/assets/img/concurrent-macro-and-standard-cell-placement-01.png)

![Concurrent placement in one sequence]({{ site.baseurl }}/assets/img/concurrent-macro-and-standard-cell-placement-02.png)

---

## Guides, regions, and fences
{: #guides-regions-and-fences }

These are placement constraints that tie a logical module to a physical area. They differ
in strictness.

![Guides and regions]({{ site.baseurl }}/assets/img/floorplanning-05.png)

| Constraint | Cells from the module | Other cells |
|:--|:--|:--|
| **Guide** | Preferred inside, may leave | May enter |
| **Region** | Must stay inside | May enter |
| **Fence** | Must stay inside | May **not** enter |

A **guide** is a soft hint. A **region** is a hard boundary for the module's own cells. A
**fence** is exclusive in both directions, which is what you want for a partition that will
be implemented separately or for a block with its own power domain.

Module guides are generated automatically from the netlist hierarchy. Their size is
computed from the number of standard cells in the module and an estimated utilisation, so
they show you roughly how much area each module will need.

![Module guides versus hard macros]({{ site.baseurl }}/assets/img/floorplanning-06.png)

The **minimum floorplan module size** setting controls how finely the tool splits the
hierarchy: any logical unit smaller than the threshold is merged into another rather than
getting its own guide. Setting it to 100, for example, avoids a mess of tiny guides for
trivial modules.

![Minimum module size]({{ site.baseurl }}/assets/img/floorplanning-07.png)

{: .tip }
> Constraints are a trade. They give you control over locality, which can help timing and
> congestion — but every constraint takes freedom away from the placer, and an
> over-constrained floorplan usually places worse than an unconstrained one. Use them where
> you have a specific reason, not by default.

---

## Blockages and halos

A **blockage** tells the tool it may not use an area.

![Types of blockages]({{ site.baseurl }}/assets/img/types-of-blockages-routing-and-placement-01.png)

| Type | Effect |
|:--|:--|
| **Hard placement blockage** | No standard cells may be placed here at all |
| **Soft placement blockage** | No cells during initial placement, but optimisation and CTS may use it |
| **Partial placement blockage** | Placement allowed up to a specified density percentage |
| **Routing blockage** | No routing on specified layers in this area |

A **partial** blockage is the subtle and useful one: instead of forbidding placement, it
caps how much of the area may be used.

![Partial blockage]({{ site.baseurl }}/assets/img/types-of-blockages-routing-and-placement-02.png)

![Creating a partial blockage]({{ site.baseurl }}/assets/img/types-of-blockages-routing-and-placement-03.png)

This is the right tool for a region you expect to be congested. Forbidding placement
outright just pushes the cells — and the congestion — somewhere else. Thinning the
placement locally relieves the pressure without relocating the problem.

{: .tip }
> A real debugging case: a floorplan with many narrow channels had congestion and broken
> timing paths. Adding a partial blockage to the single worst area simply moved the
> congestion elsewhere. Adding **soft blockages in all the channels** was what actually
> worked — because it addressed the structural cause (cells being placed in spaces too
> narrow to route) rather than one symptom.

### Halos

A **halo** (or keep-out margin) is a band around a macro where standard cells may not be
placed. It prevents cells from crowding right up against macro edges, where pin access is
poor and where routing to the macro's own pins needs room.

![Macro halo]({{ site.baseurl }}/assets/img/floorplanning-09.png)

A halo is not an absolute routing ban — a direct route to a macro pin is still allowed. What
it discourages is unrelated routing detouring through the region, which would compete with
the macro's own connections and create signal integrity problems.

Running the **finish floorplan** step adds placement blockages and halos around macros
automatically as a starting point.

![Finishing the floorplan]({{ site.baseurl }}/assets/img/floorplanning-10.png)

---

## I/O pin assignment

The netlist declares the block's ports but says nothing about where they physically sit.
Pin locations come from one of:

1. An **I/O assignment file**, read during design import or with `init_io_file`.
2. A **DEF** file containing pin placement, read with `defIn`.
3. Automatic assignment — `assignIoPins -pin *` spreads all pins around the core margin.

![I/O pin assignment]({{ site.baseurl }}/assets/img/pin-assignment-information-01.png)

![I/O file format]({{ site.baseurl }}/assets/img/pin-assignment-information-02.png)

Pin placement is a real timing decision at block level. A pin on the wrong edge means every
path using it starts with a long detour. Where you have freedom, place pins on the edge
nearest the logic that uses them — and where the block sits inside a larger chip, pin
locations are usually dictated by the integration team instead.

---

## Scripting the floorplan

Floorplanning is iterative, and you will redo it. Capture the whole thing in a
`floorplan.tcl` you can source, rather than clicking through the GUI each time.

![Sourcing a floorplan script]({{ site.baseurl }}/assets/img/floorplanning-11.png)

Before you start, run `checkDesign -netlist` to confirm every input was read in correctly.
A missing library or an unresolved reference discovered at floorplan time costs minutes; the
same problem discovered at routing costs days.

---

## Sources

- Silicon Integration Initiative / Cadence, *LEF/DEF 5.8 Language Reference* — SITE, ROW,
  and placement status (PLACED, FIXED, COVER, UNPLACED) definitions in DEF.
  <http://coriolis.lip6.fr/doc/lefdef/lefdefref/LEFSyntax.html>
- *I/O File Setup for PnR Using Innovus*, Digital System Design — I/O assignment file
  format and how it is read in. <https://digitalsystemdesign.in/i-o-file-setup-for-pnr-using-innovus/>
- C. Patel, *Standard Cell Library / Library Exchange Format*, CMPE 641, UMBC — site
  definitions and how rows derive from them.
  <https://courses.cs.umbc.edu/graduate/CMPE641/Fall08/cpatel2/slides/lect04_LEF.pdf>
- *Clock Tree Optimization Methodologies*, Semiconductor Digest — on macro-dominated
  floorplans constraining clock distribution choices.
  <https://www.semiconductor-digest.com/clock-tree-optimization-methodologies-for-power-and-latency-reduction/>
