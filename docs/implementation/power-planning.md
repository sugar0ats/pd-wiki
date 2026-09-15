---
layout: default
title: Power Planning
parent: 4. Implementation
nav_order: 2
---

# Power Planning
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## The problem

Every cell in the design needs VDD and VSS. There are millions of them, spread across the
core, and the supply arrives at a handful of points on the die edge or through a bump
array.

Metal has resistance. Current flowing through that resistance drops voltage — **IR drop** —
so a cell far from a supply source sees a lower VDD than the nominal value. A cell at lower
supply switches more slowly, which means timing closes in simulation and fails in silicon.
Push it far enough and gates stop functioning altogether.

The other failure mode is **electromigration**: wires carrying high current density
gradually degrade until they open. Both problems are decided by the geometry you build in
this step.

{: .warning }
> Consequences of poor power planning: timing degradation and outright gate failure from
> supply droop, and [electromigration]({{ site.baseurl }}/docs/signoff/physical-verification/#em-electromigration)
> failures years into the field. Neither is fixable late. Both are cheap to prevent now.

---

## The power distribution network

The standard solution is a **power grid**: a lattice of VDD and VSS conductors spanning the
core, so that every cell is close to a supply connection regardless of where it sits.

![Power grid structure]({{ site.baseurl }}/assets/img/power-planning-01.png)

The grid is built in layers, and the vocabulary is worth getting straight:

| Term | Where | Width | Purpose |
|:--|:--|:--|:--|
| **Ring** | Around the core, or around a block | Wide | Collects supply from the pads and distributes it around the perimeter |
| **Stripe** | Upper metal layers, crossing the core | Wide | Carries current from the rings into the interior |
| **Rail** | Lowest layers (M1/M2), along each row | Thin | Delivers supply to individual standard cells |

Three structural facts:

1. **The grid alternates direction by layer.** Horizontal on one layer, vertical on the
   next, matching each layer's preferred routing direction. The alternation is what makes
   it a mesh rather than a set of parallel lines.
2. **Higher layers are thicker and less resistive.** Upper metal is physically thicker, so
   it carries the bulk current with the least drop. Lower layers handle the last short hop
   to the cells.
3. **Vias connect the layers.** A stripe on M7 is useless unless there is a robust via
   stack down to the M1 rails. Via resistance is often the real bottleneck.

### Mesh versus rings and stripes

A full **power mesh** covering the whole core is the common approach in ASICs: dense,
robust, and forgiving. It also consumes a lot of routing resource on the upper layers.

The lighter alternative is **rings and stripes**: draw rings around blocks, and use stripes
only to connect those rings back to the external VDD and VSS. This uses fewer routing
resources and typically less power, at the cost of a less uniform supply.

{: .interview }
> *"How does your floorplan affect IR drop? Where would you place power straps for a
> high-density logic block?"* Macros block stripes from crossing them, so regions shadowed
> by macros — especially between two macros — are furthest from any supply path and droop
> most. For a high-density logic block you want stripes running directly over it at tighter
> pitch than elsewhere, with a good via stack down to the rails, and you want it not to be
> boxed in by macros on multiple sides.

### Macros and the grid

Running stripes over a macro can degrade that macro's own performance — the macro has its
own internal power structure and its own sensitivity to coupling. Usually you create a
routing blockage over the macro on the relevant layers so the grid routes around it
instead. Either the tool knows to do this from the macro's LEF, or you tell it.

---

## Power planning versus power routing

Worth separating, because the commands differ:

**Power planning** builds the *sources* — the rings and stripes that form the distribution
skeleton.

**Power routing** connects those sources to the *consumers* — running VDD and VSS down to
each standard cell's power pins and its **follow pins** (the rail segments that run through
the cell along the row).

![Power planning versus power routing]({{ site.baseurl }}/assets/img/power-planning-11.png)

---

## Commands

### `addRing`

Creates power rings around the core or a block.

![addRing]({{ site.baseurl }}/assets/img/power-planning-02.png)

- `-width` — the width of the ring conductor.
- `-spacing` — the gap between the VDD and VSS conductors.
- `-offset` — the horizontal/vertical offset from the boundary.

{: .warning }
> The area between the core boundary and the die edge must be large enough to hold the
> rings. If it is not, the command silently produces nothing useful. Size that margin when
> you set up the floorplan, not after.

### `addStripe`

Adds the stripes crossing the core. Pitch, width, and layer are the key parameters, and
they are where the IR-drop-versus-routing-resource trade is made.

![addStripe and editPowerVia]({{ site.baseurl }}/assets/img/power-planning-03.png)

![Stripe configuration]({{ site.baseurl }}/assets/img/power-planning-04.png)

![Stripe options]({{ site.baseurl }}/assets/img/power-planning-05.png)

![Stripe placement]({{ site.baseurl }}/assets/img/power-planning-06.png)

### `editPowerVia`

Creates the via stacks that tie the layers of the grid together. Neglecting these is a
common cause of a grid that looks right in the GUI and performs badly in analysis.

![Power vias]({{ site.baseurl }}/assets/img/power-planning-07.png)

![Via configuration]({{ site.baseurl }}/assets/img/power-planning-08.png)

### `globalNetConnect`

Connects the global power and ground *nets* to the corresponding *pins* on every instance.
Without this the cells are physically sitting on the rails but not logically connected to
them, and everything downstream — rail analysis, LVS — will complain.

### Flash PG

For a more automated flow, **Flash PG** generates the power grid from a specification file
rather than from individual commands.

![Flash PG]({{ site.baseurl }}/assets/img/power-planning-09.png)

![Flash PG file example]({{ site.baseurl }}/assets/img/power-planning-10.png)

---

## Special route (SRoute)

**SRoute** is the power-specific router. It handles the connections that ordinary signal
routing does not: rails to stripes, stripes to rings, macro power pins to the grid, and
follow-pin routing along rows.

![Special route]({{ site.baseurl }}/assets/img/power-planning-12.png)

Two options worth understanding:

**Jogging** allows a route to change orientation — running horizontally on a layer whose
preferred direction is vertical, or vice versa — to get around an obstacle. Useful, but
each jog adds resistance and consumes tracks in the wrong direction.

**Layer change** allows the route to move between layers en route.

![Splitting wide routes]({{ site.baseurl }}/assets/img/power-planning-13.png)

The figure above shows a route from a pad pin divided into several narrower parallel
strips rather than one wide conductor. Splitting improves IR drop by spreading the current
across multiple paths into the grid instead of funnelling it through a single entry point.
This is relevant for wire-bond style packaging where power enters through peripheral pads;
with flip-chip, power arrives through a distributed bump array and the problem looks quite
different.

{: .note }
> SRoute interprets pin shapes differently from the signal router, which is a common source
> of surprise when a macro's power pins are unusual shapes. Cadence support documentation
> covers the specific cases.

---

## Checking your work

Two analyses, and they are not the same thing:

- **[Rail analysis]({{ site.baseurl }}/docs/signoff/power-analysis/#rail-analysis)**
  (Power → Rail Analysis) — how robust the distribution network is. Produces an IR drop map.
- **[Power analysis]({{ site.baseurl }}/docs/signoff/power-analysis/)**
  (Power → Power Analysis) — how much power the design consumes.

Also run `verifyWireGap -wireToWire <distance>` after power planning to confirm no power
stripe has been accidentally broken or shorted.

Doing a first-pass rail analysis right after building the grid, before placement, is cheap
and tells you whether the grid geometry is remotely adequate. Waiting until signoff to
discover the answer is not cheap.

---

## Sources

- *Electromigration Concerns Grow In Advanced Packages*, Semiconductor Engineering — on how
  shorter, wider interconnects improve mean time to failure, and on temperature dependence.
  <https://semiengineering.com/electromigration-concerns-grow-in-advanced-packages/>
- *Black's Equation for MTTF Due to Electromigration*, Cadence — on current density driving
  electromigration, and the thermal runaway interaction with IR drop.
  <https://resources.system-analysis.cadence.com/blog/msa2020-blacks-equation-for-mttf-due-to-electromigration>
- C. Patel, *Standard Cell Library / Library Exchange Format*, CMPE 641, UMBC — layer
  resistance and capacitance per square, which determine grid drop.
  <https://courses.cs.umbc.edu/graduate/CMPE641/Fall08/cpatel2/slides/lect04_LEF.pdf>
- R. Amirtharajah, *CMOS Power Dissipation and Trends*, EEC 216, UC Davis — switching
  current as the source of the transient demand the grid must supply.
  <https://www.ece.ucdavis.edu/~ramirtha/EEC216/W08/lecture1_updated.pdf>
