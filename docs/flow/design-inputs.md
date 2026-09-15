---
layout: default
title: Design Inputs and File Formats
parent: 2. The Flow and Its Inputs
nav_order: 2
---

# Design Inputs and File Formats
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## What you get handed

To start place and route on a block you need, at minimum:

| # | Input | Describes | Comes from |
|:--|:--|:--|:--|
| 1 | **Gate-level netlist** | What is connected to what | Synthesis |
| 2 | **SDC constraints** | Clocks, I/O timing, exceptions | Design team / synthesis |
| 3 | **LEF** (tech + cell) | Physical shape of everything | Foundry + library vendor |
| 4 | **`.lib` (Liberty)** | Timing and power of every cell | Library vendor |
| 5 | **Technology file** | Layer stack, RC per layer | Foundry |
| 6 | **ITF / extraction tech file** | Parasitic modelling per layer | Foundry |
| 7 | **Macro / library list** | Which macros this design uses | Design team |
| 8 | **DEF** *(optional)* | Existing physical configuration | Previous run / floorplan |
| 9 | **I/O assignment file** | Where pins go | Design team / chip integration |
| 10 | **CPF / UPF** *(if low power)* | Power intent | Design team |

Missing or mismatched inputs produce failures that look like tool bugs. A `.lib`
characterised at a different operating voltage than the one you are analysing, for
instance, will give you delay numbers that are quietly and completely wrong.

---

## Gate-level netlist

The **gate-level netlist (GLN)** is a textual description, usually in Verilog, of how
standard cell and macro instances are wired together. It is the output of synthesis and
the primary functional input to physical design.

Netlists can be:

- **Flat** — everything in a single module.
- **Hierarchical** — modules instantiated inside other modules, mirroring the RTL
  structure. Hierarchy is useful for floorplanning, because you can place a module's cells
  together in a [region or guide]({{ site.baseurl }}/docs/implementation/floorplanning/#guides-regions-and-fences).

The netlist names the design's input and output **ports** and what they connect to — but
it says nothing about *where* those ports physically sit. That is the job of the I/O
assignment file or DEF.

![Design browser view of netlist hierarchy]({{ site.baseurl }}/assets/img/gate-level-netlist-01.png)

---

## SDC (Synopsys Design Constraints)

**SDC** is the timing constraint language. It is written in Tcl, generated or hand-written
alongside synthesis, and it tells every downstream timing tool what "correct" means.

Without SDC, a timing tool has no idea what frequency you are targeting, what the outside
world looks like, or which paths it is allowed to ignore. It is, in practice, the file
most likely to be the real cause of a confusing violation.

![SDC contents]({{ site.baseurl }}/assets/img/sdc-synopsys-design-constraint-01.png)

### What goes in an SDC

**Clock definitions**

- `create_clock` — defines a clock at a source: its period and waveform.
- `create_generated_clock` — defines a clock *derived* from another (divided, multiplied,
  inverted, phase-shifted). Because it is derived, changes to the master clock propagate
  automatically.

**External environment** — the chip does not exist in isolation, so you model what is
outside it:

- `set_input_delay` — how late a signal arrives at an input port relative to a clock edge
  at the external launching register.
- `set_output_delay` — how much time the external world needs after your output port
  before its own capture edge.
- `set_driving_cell` — what kind of cell is driving your inputs, so input slew is realistic.
- `set_load` — the capacitance your outputs must drive.

**Design rule constraints**

- `set_max_transition` — cap on how slow an edge may be.
- `set_max_fanout` — cap on how many loads one driver may feed.
- `set_max_capacitance` — cap on load capacitance.

**Clock non-idealities** — see [clocks and skew]({{ site.baseurl }}/docs/timing/clocks/):

- `set_clock_latency` — how long the clock takes to reach a point.
- `set_clock_uncertainty` — margin absorbing jitter, skew, and extra pessimism.

**Timing exceptions**
{: #timing-exceptions }

- `set_false_path` — a path that can never be functionally exercised, so it should be
  excluded from analysis entirely. Constraining a false path wastes optimisation effort on
  something that does not matter.
- `set_multicycle_path` — a path that is legitimately allowed more than one clock cycle to
  propagate. Tells the tool to move the capture edge instead of flagging a violation.

**Path grouping**

- `group_path` — buckets paths so the optimiser's cost function treats them separately. A
  group with a catastrophic violation will not then starve every other group of
  optimisation effort. See
  [path groups]({{ site.baseurl }}/docs/signoff/timing-closure/#path-groups).

![SDC specification summary]({{ site.baseurl }}/assets/img/sdc-synopsys-design-constraint-02.png)

{: .interview }
> *"What's the difference between a false path and a multicycle path?"* A false path is
> never exercised and is removed from analysis. A multicycle path is genuinely exercised
> but is allowed N cycles instead of one, so it is still checked — just against a
> different edge. Declaring something false when it is really multicycle means you stop
> checking a path that can actually fail.

---

## LEF (Library Exchange Format)
{: #lef-library-exchange-format }

**LEF** is an ASCII format describing the *physical* view of a library. It is used
industry-wide, alongside DEF, and is distributed as an open standard by Si2.

Critically, LEF describes cells **abstractly**. It gives pin locations and obstructions
without exposing the cell's internal implementation — exactly what a place-and-route tool
needs and nothing it does not.

![LEF file structure]({{ site.baseurl }}/assets/img/library-exchange-file-lef-01.png)

You can put everything into one LEF file, but that gets unwieldy, so it is conventionally
split in two:

**Technology LEF ("tech LEF")** — process and routing rules for the node:

- **Layer** definitions in process order, bottom to top, each with a type (routing, cut,
  masterslice, overlap), width/pitch/spacing rules, preferred direction, resistance and
  capacitance per unit square, and antenna factors.
- **Via** definitions and via rules.
- **Site** definitions (which become [rows]({{ site.baseurl }}/docs/fundamentals/cells/#sites-and-rows)).
- The **manufacturing grid**.

**Cell library LEF** — one MACRO entry per standard cell and per hard macro:

- Cell dimensions and class.
- Pin names, directions, and the shapes and layers each pin occupies.
- **Obstructions (OBS)** — regions inside the cell footprint where the router must not
  route.

![LEF cell and technology sections]({{ site.baseurl }}/assets/img/library-exchange-file-lef-02.png)

{: .note }
> **Can you edit the LEF?** The tech LEF comes from the foundry and you should treat it as
> read-only — it encodes manufacturing rules. Cell LEF is normally delivered with the
> standard cell library, but if a block is missing one (a new macro, a hard IP without an
> abstract), you can generate an abstract view from the layout using an abstract generation
> flow, e.g. `set_abstract_mode` followed by `run_abstract`.

---

## `.lib` (Liberty) timing libraries
{: #lib-liberty-timing-libraries }

Where LEF describes shape, **Liberty (`.lib`)** describes behaviour. For every cell it
holds:

- **Delay tables** — propagation delay as a function of input transition time and output
  load capacitance, usually as a 2-D lookup table.
- **Input pin capacitance**.
- **Timing checks** — setup and hold requirements for sequential cells.
- **Power data** — internal switching energy and leakage.
- **[Unateness]({{ site.baseurl }}/docs/timing/sta-basics/#unateness)** of each timing arc.

A library is characterised at one specific **PVT corner** — process, voltage, temperature.
That is why you need several of them: a slow library for setup analysis, a fast one for
hold. See [corners and modes]({{ site.baseurl }}/docs/timing/variation/#corners-modes-and-mmmc).

{: .note }
> **`.tlf`** (Timing Library Format) is an older Cadence timing library format you may see
> referenced in legacy scripts. Liberty superseded it.

---

## DEF (Design Exchange Format)

If LEF describes the *library*, **DEF** describes *this particular design*. It is the
companion format, also ASCII, also from Si2.

A DEF file carries the physical configuration of the design: die and core area, row
definitions, routing tracks, component instances and their placement coordinates and
status, I/O pin locations, net connectivity, blockages, and routed geometry.

DEF is how floorplans get saved, shared, and handed between tools. It is also the vehicle
for a few specific things:

- **I/O pin assignment** — read a DEF with `defIn` to place pins.
- **scanDEF** — a subsection describing
  [scan chains]({{ site.baseurl }}/docs/implementation/placement/#scan-chains): where each
  chain starts and ends, so that P&R can reorder it.

---

## Parasitic RC and extraction
{: #parasitic-rc-and-extraction }

Wires are not ideal. Every metal interconnect has **resistance** along its length and
**capacitance** to the substrate and to neighbouring wires. Together these are the
**parasitics**, and at advanced nodes they dominate delay — a wire's RC can easily exceed
the delay of the gate driving it.

Capacitance between adjacent wires (**coupling capacitance**) is doubly problematic: it
slows things down *and* it lets neighbouring nets interfere with each other, which is
[crosstalk]({{ site.baseurl }}/docs/signoff/signal-integrity/).

How parasitics are estimated changes through the flow:

| Stage | Model | Accuracy |
|:--|:--|:--|
| Pre-placement | **Wire load model** — statistical estimate of wire length from fanout | Crude |
| Post-placement | Virtual/early global route estimate | Rough |
| Post-global-route | Global route based extraction | Reasonable |
| Post-detail-route | **Full extraction** from actual geometry | Signoff quality |

### The extraction tool

An **extraction tool** reads the routed design and computes detailed R and C values for
every net, writing them out as a **SPEF** file (Standard Parasitic Exchange Format), which
the timing tool then consumes.

![Extraction flow]({{ site.baseurl }}/assets/img/extraction-tool-01.png)

Extraction engines sit on an accuracy-versus-runtime spectrum, and you pick according to
what stage you are at:

![Extraction options]({{ site.baseurl }}/assets/img/extraction-tool-02.png)

![Setting extraction modes]({{ site.baseurl }}/assets/img/extraction-tool-03.png)

**RC correlation** is the practice of checking that your implementation tool's extraction
agrees with your signoff extraction tool. If they disagree, you are optimising against
numbers that signoff will later reject — you want to discover that early, not the week
before tapeout.

![Benefits of RC correlation]({{ site.baseurl }}/assets/img/extraction-tool-04.png)

The extraction technology file (sometimes **ITF**, or a `.tch` extraction tech file) is the
foundry-supplied description of the layer stack that makes accurate extraction possible.

---

## SDF (Standard Delay Format)

**SDF** is an industry-standard format for carrying **cell and interconnect delay values**
between tools. Its most common use is back-annotation: you write out SDF from your timing
tool and feed it to a gate-level simulator so the simulation reflects real post-layout
delays rather than idealised ones.

![SDF]({{ site.baseurl }}/assets/img/sdf-01.png)

---

## CPF / UPF (power intent)

**CPF** (Common Power Format) and **UPF** (Unified Power Format) describe **power intent**:
the things a netlist cannot express about how power is organised. They matter for designs
with multiple supply voltages, power domains that can be switched off, level shifters,
isolation cells, and retention flops.

If a design is single-voltage and always-on, you will not see one of these files. If it is
a mobile SoC, power intent is central.

---

## Sources

- Silicon Integration Initiative / Cadence, *LEF/DEF 5.8 Language Reference* — LEF file
  organisation, technology versus cell library LEF, layer and macro statements.
  <http://coriolis.lip6.fr/doc/lefdef/lefdefref/LEFSyntax.html>
- *LEF/DEF*, Wikipedia — LEF/DEF as an Apache-licensed open standard distributed by Si2
  and used primarily by place-and-route tools. <https://en.wikipedia.org/wiki/LEF/DEF>
- C. Patel, *Standard Cell Library / Library Exchange Format*, CMPE 641, UMBC — LEF
  sections, layer attributes, resistance and capacitance per square.
  <https://courses.cs.umbc.edu/graduate/CMPE641/Fall08/cpatel2/slides/lect04_LEF.pdf>
- *Synopsys Design Constraints (SDC) File in VLSI*, Team VLSI — SDC command categories and
  syntax. <https://teamvlsi.com/2020/05/sdc-synopsys-design-constraint-file-in.html>
- R. Robucci, *Timing Analysis*, UMBC — on SDF as the delay interchange format alongside a
  post-place-and-route netlist.
  <https://eclipse.umbc.edu/robucci/cmpeRSD/Lectures/Lecture13__TimingAnalysis/>
- *Gate Level Netlist*, Microelectronics Institute of Seville (IMSE-CNM) teaching notes.
  <http://www2.imse-cnm.csic.es/elec_esi/asignat/MHCAD/tema1_digital/Gate_level_netlist.html>
