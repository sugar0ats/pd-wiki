---
layout: default
title: Clock Tree Synthesis
parent: 4. Implementation
nav_order: 4
---

# Clock Tree Synthesis
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## What CTS is

**Clock tree synthesis** builds the physical network that carries the clock from its source
to the clock pin of every sequential element in the design, and balances the arrival times
across those endpoints.

Up to this point the clock has been
[ideal]({{ site.baseurl }}/docs/timing/clocks/#the-ideal-clock-tree) — assumed to arrive
everywhere at once with no delay. CTS replaces that fiction with real buffers and real
wires. Afterwards the clock is **propagated**, and timing analysis sees actual
[insertion delay]({{ site.baseurl }}/docs/timing/clocks/#clock-latency-insertion-delay) and
actual [skew]({{ site.baseurl }}/docs/timing/clocks/#skew).

The clock network is unlike any other net in the design. It is the highest-fanout net, it
switches every cycle so it dominates dynamic power, and every flip-flop's timing depends on
it. It gets special treatment throughout.

Three broad styles of clock path:

1. **Ideal** — clock source to every sink is exactly equal. A model, not a buildable thing.
2. **Traditional skew-balanced** — source to sinks equal within a tolerance.
3. **Useful skew** — deliberately unbalanced, with skew used as a resource to fix timing.

---

## Why zero skew is not the goal

A perfectly balanced tree sounds ideal. It is not, for three reasons:

**1. IR drop.** If every flop's clock edge arrives simultaneously, every flop switches
simultaneously, and the whole block draws its peak current in the same instant. That
transient droop degrades timing across the block — so enforcing zero skew can cause the
problem it was meant to prevent.

**2. Area and power.** Perfect balance is bought with a large number of clock buffers. More
cells means more area and more leakage. Meanwhile a tighter skew requirement pushes
insertion delay up, and higher insertion delay means higher clock network power.

**3. Wasted opportunity.** Skew is usable. Deliberately delaying a capture clock moves
margin from a path that has slack to a path that does not.

![Why not zero skew]({{ site.baseurl }}/assets/img/cts-03.png)

![Clock tree cost]({{ site.baseurl }}/assets/img/cts-04.png)

### Useful skew and time borrowing

Take a path with 5 ns of combinational delay in a 4 ns clock period. It violates setup by
1 ns.

![Useful skew example]({{ site.baseurl }}/assets/img/cts-05.png)

Rather than making the logic 20% faster, delay the **capture** clock by 1 ns. The path now
has 5 ns available and passes — borrowed from the next stage, which now has 3 ns instead of
4. This works whenever the next stage has slack to lend.

The mirror-image move works too: remove buffers from the **launch** clock path so data is
launched earlier.

![Launch path adjustment]({{ site.baseurl }}/assets/img/cts-06.png)

{: .warning }
> Useful skew relocates a problem rather than deleting it. Borrow from a stage with no slack
> and you have simply moved the violation. This is why modern flows use concurrent clock and
> datapath optimisation, which reasons about the whole graph at once, rather than balancing
> a tree and then fixing timing separately.

---

## Clock tree architectures

![CTS algorithms]({{ site.baseurl }}/assets/img/cts-07.png)

![Architecture comparison]({{ site.baseurl }}/assets/img/cts-08.png)

### Conventional (single-source) CTS

A single clock root branching out through buffers to all sinks. Simplest to build, lowest
clock switching power, and the default for most designs — particularly lower frequency ones
and designs with multiple clock domains.

Its weakness is **OCV penalty**. Because the tree diverges right at the clock source, the
launch and capture paths to two different flops share very little common path, so
[on-chip variation]({{ site.baseurl }}/docs/timing/variation/#ocv-on-chip-variation) derating
applies over almost the whole depth of both paths. Single-source CTS carries the largest OCV
penalty of the architectures.

### Clock mesh

A grid of clock wire shorted together, driven by many buffers, with sinks tapping off
wherever they happen to be.

Meshes give excellent skew and are highly tolerant of on-chip variation, which makes them
the choice for high-frequency single-clock-domain designs — CPUs and GPUs. The logic depth
*after* the mesh is very shallow, so most of the insertion delay is in the shared path from
the root up to the mesh, and that shared path means the OCV penalty is minimal.

The costs are severe: a mesh consumes a lot of routing resource, is highly capacitive, and
**cannot be clock-gated**, so it burns power continuously.

### H-tree

A geometrically structured tree whose branches are length-matched, giving very low skew by
construction rather than by iterative balancing.

Classic H-trees need **regular, unblocked rectangular area** and a power-of-two number of
sinks, which real floorplans rarely provide. **Flexible H-trees** relax the strict geometry
while keeping the length-matching idea, and are far more usable in a macro-dominated
floorplan than a mesh is.

H-tree construction in practice: place high-drive clock cells at predefined locations, route
the clock nets as straight as possible to minimise skew, keep them on upper metal layers,
and apply NDRs plus `don't touch` so nothing later disturbs them.

### Multi-tap / multi-source CTS

A hybrid, and increasingly the mainstream choice for large designs. The block is divided
into partitions, each with its own **tap point** acting as a local clock source. An H-tree
distributes the clock from the root to the tap points; a conventional tree is then built
below each tap.

![Multi-tap CTS]({{ site.baseurl }}/assets/img/cts-09.png)

This gets much of the mesh's skew and OCV benefit — the common path now extends all the way
to the tap points — without the mesh's power and routing cost.

| | Conventional | Mesh | H-tree | Multi-source |
|:--|:--|:--|:--|:--|
| Skew | Largest | Smallest | Very small | Small |
| Power | Lowest | Highest | High | Moderate |
| Routing resource | Low | Very high | High | Moderate |
| OCV penalty | Largest | Smallest | Small | Small |
| Clock gating | Yes | No | Yes | Yes |
| Floorplan tolerance | High | Moderate | Low (strict), moderate (flexible) | Moderate |

{: .interview }
> *"What's the difference between an H-tree and a mesh clock network? Which uses more
> power?"* The mesh. A mesh is a shorted grid driven from many points — excellent skew and
> OCV tolerance, but highly capacitive, resource-hungry, and it cannot be gated, so it
> dissipates continuously. An H-tree is still a tree, so it can be gated and uses less
> power, and it achieves low skew through matched branch lengths — but it wants regular
> unblocked area, which real floorplans with macros often will not give you.

---

## Clock trees versus skew groups

A distinction that causes a lot of confusion:

**Clock tree** — the *physical* object. Which nets exist, how they are routed, where the
buffers are.

**Skew group** — a *constraint* abstraction. A set of sinks the tool is asked to balance
against one another, with skew and insertion delay targets. It tells the tool what to
optimise, not what to build.

![Clock tree versus skew group]({{ site.baseurl }}/assets/img/cts-10.png)

![Skew group definition]({{ site.baseurl }}/assets/img/cts-11.png)

Skew groups are created automatically from the SDC. They **can overlap**: if two clock
signals feed a mux and the downstream logic may run on either, the flops below that mux
belong to both groups.

![Overlapping skew groups]({{ site.baseurl }}/assets/img/cts-12.png)

![Skew group configuration]({{ site.baseurl }}/assets/img/cts-13.png)

### Clock tree spec files

![Clock tree spec]({{ site.baseurl }}/assets/img/cts-14.png)

The clock tree specification can be generated automatically by `clock_opt_design`, or
created ahead of time with an explicit command so you can inspect and adjust it.

**Ignore pins.** Consider a mux with clock `ck` on input 0 and gated clock `gck` on input 1.
These are different clock signals and the tool must not try to balance one against the
other. Setting the **ignore pin** attribute on the appropriate pin tells it so.

![Ignore pins on a mux]({{ site.baseurl }}/assets/img/cts-15.png)

{: .warning }
> If you need to change clock tree attributes after the spec file exists, **do not edit the
> file**. Run commands that override the properties instead. Hand-edited spec files are
> regenerated out from under you by the next `clock_opt_design`.

![Overriding clock tree properties]({{ site.baseurl }}/assets/img/cts-16.png)

### How SDC becomes CCOpt constraints

![SDC to CCOpt]({{ site.baseurl }}/assets/img/cts-17.png)

---

## Running CTS

{: .tip }
> Run `check_design -type cts` **first**. It verifies the clock tree definition is
> structurally correct before you spend an hour optimising a tree built on a broken
> specification.

![CTS commands]({{ site.baseurl }}/assets/img/cts-18.png)

![CTS command sequence]({{ site.baseurl }}/assets/img/cts-19.png)

![CTS flow]({{ site.baseurl }}/assets/img/cts-20.png)

### `set_ccopt_property`

The main configuration mechanism. A few properties worth knowing:

![set_ccopt_property]({{ site.baseurl }}/assets/img/cts-21.png)

**`cell_halo_x` / `cell_halo_y`** — padding around clock tree cells. Clock cells often have
high drive strength and draw large currents; placing them shoulder to shoulder creates
local IR problems and can violate spacing rules. Halos keep them apart.

**`auto_limit_insertion_delay_factor`** — caps the insertion delay to a sink. Insertion
delay can drift during CTS as the tool applies useful skew and time borrowing; this puts a
ceiling on it, either per sink or across all sinks. Worth using, because
[large insertion delay amplifies OCV pessimism]({{ site.baseurl }}/docs/timing/clocks/#clock-latency-insertion-delay).

![Insertion delay limits]({{ site.baseurl }}/assets/img/cts-22.png)

### Reporting

| Command | Reports |
|:--|:--|
| `report_clock_timing -type skew` | Skew across the tree |
| `report_ccopt_clock_tree_structure` | The structure the tool built |
| `show_ccopt_cell_name_info` | Naming conventions used for clock tree cells |

### Effort levels

![Flow effort and CTS]({{ site.baseurl }}/assets/img/cts-24.png)

Setting the flow effort changes how hard CTS works and which transformations it is willing
to attempt, trading runtime against quality.

### What `clock_opt_design` actually does

![clock_opt_design steps]({{ site.baseurl }}/assets/img/cts-25.png)

![clock_opt_design detail]({{ site.baseurl }}/assets/img/cts-26.png)

It is a sequence: build the tree, route it, then optimise datapath timing with the
propagated clock in place.

### Timing after CTS

![Timing impact of CTS]({{ site.baseurl }}/assets/img/cts-23.png)

Expect the numbers to move. The clock now has real delay and real skew, and paths that were
comfortable under an ideal clock may not be.

---

## Routing the clock tree

Clock nets are divided into three groups, and the distinction exists so you can route them
differently:

![Top, trunk, and leaf nets]({{ site.baseurl }}/assets/img/cts-27.png)

- **Top** — from the clock root down to the first level of branching. Highest current, most
  critical to skew.
- **Trunk** — the intermediate distribution.
- **Leaf** — the final connections to sink pins.

[Non-default routing rules]({{ site.baseurl }}/docs/implementation/routing/#ndrs-non-default-rules)
can be applied per group — typically wider wires and extra spacing on top and trunk nets,
where the cost is justified, and default rules on the far more numerous leaf nets.

![NDRs on clock net groups]({{ site.baseurl }}/assets/img/cts-28.png)

Clock nets are also prime candidates for
[shielding]({{ site.baseurl }}/docs/implementation/routing/#shielding), since a glitch on a
clock line is a functional failure rather than just a timing perturbation.

---

## Optimising CTS at smaller nodes

![CTS optimisation tips]({{ site.baseurl }}/assets/img/cts-29.png)

![More CTS tips]({{ site.baseurl }}/assets/img/cts-30.png)

The recurring themes:

- **Maximise common path.** The more of the clock path that launch and capture flops share,
  the more OCV variation cancels between them. This is the single biggest lever on clock
  OCV pessimism, and the main argument for multi-source architectures.
- **Keep insertion delay down.** It costs power and amplifies variation.
- **Support clock gating.** Gating is the largest available clock power saving, so do not
  pick an architecture that forbids it without a strong reason.
- **Route on upper layers.** Lower resistance, less coupling, better skew control.

### Flexible H-tree with multi-tap sources

![Flexible H-tree flow]({{ site.baseurl }}/assets/img/cts-31.png)

{: .interview }
> *"How does clock gating save power? Can it cause timing violations?"* It stops the clock
> toggling to registers that are not doing useful work, removing both their switching power
> and the clock network power below the gate — the largest single dynamic power saving
> available. It can absolutely cause timing problems: the enable signal into the gate has
> its own setup and hold requirement relative to the clock, and it is usually tight. Tools
> create a dedicated `reg2clkgate` path group precisely because these paths need watching.

---

## Sources

- *Ultimate Guide: Clock Tree Synthesis*, AnySilicon — comparison of single-point CTS, clock
  mesh, and multi-source CTS, including OCV penalty, gating, and routing resource
  trade-offs. <https://anysilicon.com/clock-tree-synthesis/>
- *Clock Tree Optimization Methodologies for Power and Latency Reduction*, Semiconductor
  Digest — on tight skew requirements increasing insertion delay and clock power, common
  path reducing OCV impact, and H-tree construction practice.
  <https://www.semiconductor-digest.com/clock-tree-optimization-methodologies-for-power-and-latency-reduction/>
- A. B. Kahng et al., *Optimal Generalized H-Tree Topology and Buffering for High-Performance
  Clocking*, UCSD VLSI CAD Laboratory — on strict H-trees achieving minimum skew at the cost
  of power, buffer area, and wirelength. <https://vlsicad.ucsd.edu/Publications/Journals/j128.pdf>
- D. Mathew et al., *A Comparative Study on Minimum Skew Clock Tree Distribution
  Algorithms*, J. Phys. Conf. Ser. 1916 012125 — measured latency, clock power, wirelength,
  buffer count, and skew across conventional, mesh, and multisource CTS.
  <https://www.researchgate.net/publication/351921926>
- *Timing-Driven Variation-Aware Synthesis of Hybrid Mesh/Tree Clock Distribution
  Networks*, University of Rochester — on hybrid mesh/tree topologies reducing metal area
  and power. <https://hajim.rochester.edu/ece/sites/friedman/papers/Integration_13.pdf>
