---
layout: default
title: Placement
parent: 4. Implementation
nav_order: 3
---

# Placement
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## What placement does

**Placement** assigns every standard cell a legal position in a row. It runs after the
planning stage — macros placed, power grid built — and before
[clock tree synthesis]({{ site.baseurl }}/docs/implementation/cts/).

![Placement in the flow]({{ site.baseurl }}/assets/img/placement-01.png)

It is not just a geometry problem. Placement decides wire lengths, and wire lengths decide
delay, congestion, and power. Modern placers therefore interleave **placement** with
**logic optimisation**: place, look at the resulting timing, resize cells and insert
buffers, place again. The single command `place_opt_design` runs this loop.

---

## The three phases

![Placement phases]({{ site.baseurl }}/assets/img/placement-02.png)

**1. Global placement.** An initial, approximate arrangement. Cells are spread across the
core to minimise total wire length and total negative slack, without worrying yet about
whether each cell sits exactly on a legal site. Cells may overlap at this stage.

**2. Incremental placement.** The engine refines the result, improving wire density and
reducing congestion hotspots.

**3. Detail placement.** Cells are **legalised** — snapped onto actual sites in actual rows,
with no overlaps. The engine ensures the result is DRC clean and that pin access is
workable.

That last point matters more than it used to. At advanced nodes, whether the router can
physically reach a cell's pins constrains placement more tightly than whether the cell
geometrically fits.

{: .interview }
> *"Why does pin density matter more than cell density in advanced nodes?"* Because
> geometric area stopped being the binding constraint. Cells shrank faster than the lowest
> metal layers' pitch did, so a legal, comfortably-spaced placement can still be unroutable
> because there are not enough M1/M2 tracks to reach every pin in that neighbourhood. You
> can have 60% cell density and a region that cannot be routed.

---

## Placement commands

### `setPlaceMode`

Configures how placement behaves, and must be called **before** running placement
optimisation. Among many other things it controls how flip-flops are handled, which matters
because flop placement determines what the clock tree will have to do.

![setPlaceMode]({{ site.baseurl }}/assets/img/placement-04.png)

![Place mode options]({{ site.baseurl }}/assets/img/placement-05.png)

`getPlaceMode` reports the current settings.

### `place_connected`

Forces cells that are logically between specified objects to be placed physically near an
"attractor".

![place_connected]({{ site.baseurl }}/assets/img/placement-03.png)

In the example shown, everything logically between the attractor `RAM1` and the cells
`C4`/`C6` is pulled physically close to `RAM1`. Useful when you know a group of logic
belongs next to a specific macro and the placer's global cost function is not seeing it.

### Instance space groups

![Instance space groups]({{ site.baseurl }}/assets/img/placement-06.png)

These solve a specific problem: a set of cells that cannot be placed adjacent to each other
because doing so would create a spacing DRC violation — for example, cells that cannot
safely share the same well. The space group tells the tool to keep them apart.

### `place_opt_design`

The main event. Runs placement and optimisation interleaved.

![place_opt_design]({{ site.baseurl }}/assets/img/placement-07.png)

Note what it optimises: `place_opt_design` targets **setup** only. Hold fixing comes later,
after CTS, because hold fixes add delay and there is no point adding delay before the clock
tree tells you how much you actually need. See
[timing closure]({{ site.baseurl }}/docs/signoff/timing-closure/#which-command-optimises-what).

### Reporting

![Reporting placement density]({{ site.baseurl }}/assets/img/placement-08.png)

`reportPlacementDensity` and its siblings report pin density, violations, and related
metrics. Read them before moving on.

### Early CTS during placement

![Early CTS]({{ site.baseurl }}/assets/img/placement-09.png)

**Early CTS** — also called pre-CTS or clock-aware placement — builds a *virtual* model of
the clock tree before the real one exists. It estimates what the clock latency and skew
will be, and where clock buffers are likely to end up.

This makes placement clock-aware. The placer can position flip-flops in clusters that will
be cheap to reach with a clock tree, rather than scattering them and leaving CTS to solve
an unnecessarily hard problem. It costs runtime now and saves considerably more later.

![Early CTS benefits]({{ site.baseurl }}/assets/img/cts-01.png)

![Clock-aware placement]({{ site.baseurl }}/assets/img/cts-02.png)

---

## Debugging placement

![Debugging placement]({{ site.baseurl }}/assets/img/placement-10.png)

If the tool still reports placement violations after optimisation has finished, the usual
moves are:

1. Raise the **placement effort** to high and rerun.
2. Add **cell padding** to spread cells out.
3. Ask the tool to **spread out hotspots** explicitly.
4. If congestion is global rather than local, go back to the floorplan — expand the core or
   move macros. No placement setting fixes a floorplan that is too tight.

---

## Cell padding

**Cell padding** reserves empty space around a placed cell, beyond its actual footprint.

![Cell padding]({{ site.baseurl }}/assets/img/cell-padding-01.png)

Two reasons to use it:

**Electrical.** Some cells — I/O buffers, high drive strength cells, clock buffers — draw
large currents. Packing other cells tight against them causes local IR problems for those
neighbours. Padding keeps a buffer zone.

**Congestion.** A region with high cell density will have high pin density and high routing
demand. Padding the cells there thins the placement locally and gives the router room,
without moving the logic somewhere less appropriate.

Padding is preventive. Applied before placement it shapes the result; applied afterwards it
forces a re-placement of the affected region.

---

## Spare cells

[Spare cells]({{ site.baseurl }}/docs/fundamentals/cells/#spare-cells) are placed during
this stage, distributed through free space where standard cells are not present.

![Spare cells]({{ site.baseurl }}/assets/img/spare-cells-01.png)

They may be any standard cell type, and the mix is chosen to cover the kinds of fix you
might plausibly need later. Their purpose is **post-mask ECO**: once the base silicon layers
are committed, rewiring spare cells through metal-only changes is the only repair available.

Spares may already be present in the netlist if synthesis was told to insert them. If not,
they can be inserted during physical design.

---

## Design for test structures

Test logic is inserted during synthesis but **placed** here, and it has physical
requirements that interact with the rest of placement.

### Scan chains
{: #scan-chains }

A **scan chain** stitches the design's flip-flops into a long shift register. In test mode,
state can be shifted in, a functional cycle applied, and the resulting state shifted out and
compared against expected values. It is the standard way to get observability into a
manufactured chip that has almost no external pins relative to its internal state.

![Scan chain]({{ site.baseurl }}/assets/img/scan-chain-01.png)

Scan insertion happens at synthesis. What physical design does is **reorder** the chain.

![Scan reordering]({{ site.baseurl }}/assets/img/scan-chain-02.png)

The logical order of flops within a chain does not matter functionally — data shifts through
all of them either way. But the *routing* cost depends entirely on the order. A chain
stitched in netlist order will zigzag across the block; reordered to follow physical
proximity, the same chain costs a fraction of the wire. If the synthesised order is 1→2→3
and the placed positions make 1→3→2 shorter, reorder it.

Commands:

| Command | Purpose |
|:--|:--|
| `specifyScanCell` | Identify scan cells |
| `specifyScanChain` | Define a chain |
| `setScanReorderMode` | Configure reordering behaviour |
| `scanReorder` | Reorder chains for routability |
| `deleteScanChain` | Remove a chain definition |

![Scan chain commands]({{ site.baseurl }}/assets/img/scan-chain-03.png)

`place_opt_design` already performs scan reordering, so the explicit commands are for when
you need finer control.

**Where the information lives.** Scan chain definitions are carried in the design's DEF, in
a subsection called **scanDEF**. It specifies where each chain starts and ends; it
deliberately does not fix the intermediate order, precisely so that P&R is free to reorder.
Synthesis tools such as Genus can generate a scanDEF.

![scanDEF format]({{ site.baseurl }}/assets/img/scan-chain-04.png)

![scanDEF example]({{ site.baseurl }}/assets/img/scan-chain-05.png)

![Writing out scan chains]({{ site.baseurl }}/assets/img/scan-chain-06.png)

{: .warning }
> Scan chains are a notorious source of **hold violations**. Adjacent scan flops may end up
> physically very close with almost no combinational delay between them, and chains often
> cross clock domains. The standard remedy is
> [lockup latches]({{ site.baseurl }}/docs/timing/clocks/#lockup-latches), which add a
> guaranteed half cycle of delay.

### JTAG

**JTAG** is a boundary-scan debug and test interface — a structure that lets external
equipment observe and drive the chip's I/O so that actual versus expected values can be
compared.

![JTAG]({{ site.baseurl }}/assets/img/jtag-01.png)

Because it interacts with the chip's boundary, **JTAG cells are placed near the core
boundary**, close to the I/O they serve.

![JTAG placement commands]({{ site.baseurl }}/assets/img/jtag-02.png)

---

## Sources

- Silicon Integration Initiative / Cadence, *LEF/DEF 5.8 Language Reference* — DEF
  COMPONENTS, placement status, and the SCANCHAINS section.
  <http://coriolis.lip6.fr/doc/lefdef/lefdefref/LEFSyntax.html>
- C. Patel, *Standard Cell Library / Library Exchange Format*, CMPE 641, UMBC — spacing and
  width rules affecting legal cell placement.
  <https://courses.cs.umbc.edu/graduate/CMPE641/Fall08/cpatel2/slides/lect04_LEF.pdf>
- *Lockup Latch*, SemiconShorts — hold violations on scan paths and domain crossings.
  <https://semiconshorts.com/2022/12/31/lockup-latch/>
- *Ultimate Guide: Clock Tree Synthesis*, AnySilicon — on clock-aware placement and
  register clustering ahead of CTS. <https://anysilicon.com/clock-tree-synthesis/>
