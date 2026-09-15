---
layout: default
title: Cells, Macros, and Sites
parent: 1. Fundamentals
nav_order: 2
---

# Cells, Macros, and Sites
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## Why pre-built cells exist

A physical design engineer does not draw transistors. Drawing transistors by hand for a
block with a few million gates is not a schedule anyone would sign up for, and the layout
rules at modern nodes are far too intricate to get right by hand at that scale.

Instead, the foundry and library vendor do that work once. They hand you a **standard cell
library**: a catalogue of pre-drawn, pre-characterised logic blocks that are guaranteed to
be manufacturable and that are guaranteed to fit together. Your job is to choose from the
catalogue and arrange the pieces.

---

## Standard cells

A **standard cell** is a reusable, pre-laid-out building block implementing one small
logic function — inverters, buffers, NAND, NOR, AOI/OAI combinations, multiplexers,
flip-flops, latches, and so on.

Every standard cell in a library shares a **fixed height** (or an integer multiple of it)
so that cells can be dropped into rows like books onto a shelf. Width varies with
complexity and drive strength.

Each cell typically comes in several variants of the same function:

- **Drive strength** — `NAND2_X1`, `NAND2_X2`, `NAND2_X4`. Larger drive means wider
  transistors, lower output resistance, faster switching into a given load, but more area
  and more input capacitance presented backwards.
- **Threshold voltage** — LVT / SVT / HVT, as described in
  [MOSFETs and CMOS logic]({{ site.baseurl }}/docs/fundamentals/mosfets-and-cmos/#threshold-voltage-flavours-vt).

The tool relies on these variants constantly. Almost every timing or power fix it applies
is some combination of *swap this cell for a different variant*, *insert a buffer*, or
*move the cell*.

### The two views of a cell

A place-and-route tool never sees a cell's transistors. It sees two abstractions:

| View | File | Contains |
|:--|:--|:--|
| **Physical abstract** | [LEF]({{ site.baseurl }}/docs/flow/design-inputs/#lef-library-exchange-format) | Cell dimensions, pin names, pin shapes and layers, internal blockages (obstructions) |
| **Timing/power model** | [`.lib` (Liberty)]({{ site.baseurl }}/docs/flow/design-inputs/#lib-liberty-timing-libraries) | Delay as a function of input slew and output load, input capacitance, leakage, internal power |

The LEF abstract deliberately **omits the internal implementation**. It gives the router
what it needs — where the pins are, where it must not route — and nothing more.

---

## Macros

A **macro** (or **hard macro**, or **block**) is a large pre-built component you place as a
single unit: SRAMs, register files, PLLs, analog blocks, I/O cells, and third-party IP.

Macros differ from standard cells in every way that matters to a floorplan:

- They are **much larger** and do not fit in standard cell rows.
- They are placed **individually and deliberately**, usually early, by a human or by a
  macro placer — not shuffled around by the detailed placer.
- They create **obstructions**: regions where standard cells cannot be placed and where
  some metal layers cannot be routed.
- Their placement dominates **congestion** and **IR drop** for the whole block.

Macro placement is the single highest-leverage decision in a floorplan, which is why it
gets its own page: [floorplanning]({{ site.baseurl }}/docs/implementation/floorplanning/).

---

## Physical cells

**Physical cells** implement no logic. They exist to make the layout legal, manufacturable,
and electrically sound. They are inserted by the tool at defined points in the flow rather
than coming from the netlist.

| Cell type | Purpose | Usually added |
|:--|:--|:--|
| **Well tap** | Ties the n-well and substrate to VDD/VSS at regular intervals, preventing latch-up | Before placement |
| **End cap / boundary** | Terminates the ends of rows and the edges of blocks so well and implant layers close cleanly | Before placement |
| **Filler** | Fills leftover gaps between placed cells to keep implant and well layers continuous | After placement, and again after ECO |
| **Decap** | Adds local decoupling capacitance to steady the supply during switching | After placement |
| **Antenna diode** | Provides a discharge path for charge accumulated during etch | During/after routing, to fix [antenna violations]({{ site.baseurl }}/docs/signoff/physical-verification/#antenna-violations) |
| **Tie-hi / tie-lo** | Safely supplies a constant 1 or 0 to a cell input without tying it directly to a rail | After placement |
| **Spare cell** | Unused logic scattered through the design, available for post-mask fixes | Placement (see below) |

![Innovus commands for adding physical cells]({{ site.baseurl }}/assets/img/physical-cells-01.png)

### Spare cells

**Spare cells** deserve special mention because they are a deliberate insurance policy.
They are ordinary standard cells — gates, flops, whatever mix the team chooses —
distributed through the design in otherwise empty space, with their inputs tied off and
their outputs unused.

The point is **post-mask repair**. Base layers (the transistor layers) are the most
expensive masks to respin. If a functional bug is found after those layers have been
committed, you may still be able to fix it by changing only the metal layers: rewire some
existing spare cells into the circuit and route around the problem. No new transistors are
fabricated, so only the cheap masks change.

Spare cells can be present in the netlist already (a synthesis option), or inserted during
physical design if they are not. They are consumed during a **post-mask
[ECO]({{ site.baseurl }}/docs/signoff/timing-closure/#eco-engineering-change-order)**.

---

## Sites and rows

The floorplan's core area is not a free canvas. It is divided into a grid.

A **site** is the smallest placement unit — the atomic tile that any cell's footprint must
be a whole-number multiple of. Site dimensions are defined in the
[technology LEF]({{ site.baseurl }}/docs/flow/design-inputs/#lef-library-exchange-format).

A **row** is a horizontal strip one site tall, running across the core. Standard cells are
placed into rows; a cell's width is always an integer number of sites, so cells snap
cleanly side by side with no wasted sliver.

![Sites and rows]({{ site.baseurl }}/assets/img/sites-and-rows-01.png)

Rows come in types. Standard cell rows hold standard cells; I/O rows hold I/O cells. The
tool places into whichever row type matches the cell.

Adjacent rows are usually **flipped and abutted** so that neighbouring rows share a power
rail — the VDD rail at the top of one row is the same physical wire as the VDD rail at the
top of the row above it, mirrored. This halves the number of rails needed. It is also why
cells have a fixed height: the rails have to line up.

---

## Routing tracks

Where rows organise *placement*, **routing tracks** organise *routing*. A track is a
preferred centreline for a wire on a given metal layer, spaced at the layer's pitch and
running in that layer's preferred direction. Tracks are defined by the technology LEF.

![Routing tracks]({{ site.baseurl }}/assets/img/routing-tracks-01.png)

Two things follow from tracks:

1. **Pin access.** A cell pin is only usable if a track can actually reach it. On the
   lowest layers (M1, M2) this is a real constraint, and at advanced nodes **pin density**
   — how many pins are crammed into an area and whether tracks can reach them all — is
   often a harder limit than raw cell density.
2. **Capacity estimation.** Tracks let the tool count how many wires can physically cross
   a given region. Global routing divides the core into **GCells** and compares demand
   against track capacity in each one; exceeding it is
   [congestion]({{ site.baseurl }}/docs/implementation/routing/#congestion).

---

## Utilisation

**Utilisation** is the fraction of available area that is occupied. It is the first number
anyone asks about a floorplan, and it is easy to quote the wrong one.

| Metric | Numerator | Note |
|:--|:--|:--|
| **Core utilisation** | Standard cells **and** macros | Over the whole core area |
| **Standard cell utilisation** | Standard cells only | Often what people mean casually |
| **Effective utilisation** | Accounts for area removed by blockages and macro obstructions | The number that predicts routability |

![Types of utilisation]({{ site.baseurl }}/assets/img/utilization-01.png)

A common starting point is around **70%** core utilisation, then tightened or loosened
based on what the design tells you. The reason to leave headroom is that placement is not
the last thing that happens: optimisation needs somewhere to put buffers, upsized cells,
clock tree cells, and ECO fixes. Packing a block to 90% at placement and then discovering
the router and the optimiser have nowhere to work is a very expensive lesson.

![Reducing utilisation systematically]({{ site.baseurl }}/assets/img/utilization-02.png)

{: .tip }
> A useful rule of thumb from the Innovus flow guidance: leave roughly **5–7%** of your
> targeted final utilisation free specifically for optimisation. See
> [timing closure]({{ site.baseurl }}/docs/signoff/timing-closure/).

---

## Sources

- Silicon Integration Initiative / Cadence, *LEF/DEF 5.8 Language Reference* — definitions
  of SITE, ROW, TRACKS, MACRO, and OBS statements.
  <http://coriolis.lip6.fr/doc/lefdef/lefdefref/LEFSyntax.html>
- C. Patel, *Standard Cell Library / Library Exchange Format*, CMPE 641 Advanced VLSI
  Design, UMBC — LEF sections and the abstract view of a cell.
  <https://courses.cs.umbc.edu/graduate/CMPE641/Fall08/cpatel2/slides/lect04_LEF.pdf>
- *LEF/DEF*, Wikipedia — on LEF/DEF as an open standard distributed by Si2.
  <https://en.wikipedia.org/wiki/LEF/DEF>
