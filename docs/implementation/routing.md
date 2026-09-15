---
layout: default
title: Routing
parent: 4. Implementation
nav_order: 5
---

# Routing
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## The routing flow

**Routing** connects every net's pins with actual metal. It runs after
[CTS]({{ site.baseurl }}/docs/implementation/cts/), and it happens in stages because solving
the whole problem exactly in one pass is computationally hopeless.

```
Global route  →  Detail route  →  Surgical repair
```

![Routing flow]({{ site.baseurl }}/assets/img/detail-route-04.png)

**Global route** plans approximate paths. The core is divided into **GCells** — rectangular
tiles, sized from the sites and rows — and the global router decides which sequence of
GCells each net passes through, without committing to exact tracks. This is fast, and it is
enough to estimate congestion and parasitics.

**Detail route** commits. Every net is assigned specific tracks, specific layers, and
specific vias, and the result must be DRC clean.

**Surgical repair** cleans up what detail route could not resolve, including
[antenna fixes]({{ site.baseurl }}/docs/signoff/physical-verification/#antenna-violations)
and remaining DRC violations.

You can run the whole thing with `route_opt_design`, or drive the pieces explicitly:

```tcl
routeDesign
routeDesign -trackOpt
optDesign -postRoute
```

![Route commands]({{ site.baseurl }}/assets/img/detail-route-05.png)

### Router types

![Router types in Innovus]({{ site.baseurl }}/assets/img/route-01.png)

**Early Global Route (EGR)** is the quick estimator, used during placement and optimisation
to predict congestion and wire parasitics. It is fast and approximate — good enough to drive
placement decisions, **not** good enough to run signal integrity analysis on.

**NanoRoute** is the detail router.

`setDesignMode` sets the top and bottom layers available for routing. `setRouteMode`
configures routing effort, direction preferences, and behaviour for groups of nets.

![setRouteMode]({{ site.baseurl }}/assets/img/route-03.png)

---

## Congestion
{: #congestion }

**Congestion** is demand for routing resources exceeding supply in some region. It is the
central problem of routing and usually the thing that sends you back to the floorplan.

### Measuring it

**Overflow.** Within a GCell, the number of available routing tracks is known. If more nets
need to cross that GCell than there are tracks, the GCell is in overflow.

**Hotspots.** Rather than looking at individual GCells, hotspot analysis looks at the
distribution of overflow and how overflowing GCells connect to each other. A cluster of
mildly overflowing GCells is a much worse problem than a single severe one, and hotspot
metrics capture that — they are generally the better indicator of overall routability.

![Hotspots]({{ site.baseurl }}/assets/img/congestion-01.png)

`reportCongestion` produces the numbers.

![Congestion concepts]({{ site.baseurl }}/assets/img/congestion-02.png)

### Reading the congestion table

![Congestion table]({{ site.baseurl }}/assets/img/congestion-06.png)

A rough scale for overcongestion percentage:

| Overcongestion | Outlook |
|:--|:--|
| Below 2% | Easy to route |
| 2–6% | Difficult |
| Above 6% | Expect many DRC violations |

{: .warning }
> The table tells you *how much* congestion there is, not *where*. Two designs with
> identical overcongestion percentages can have completely different prognoses depending on
> whether the congestion is diffuse or concentrated. Always look at the congestion map as
> well.

### The congestion map

![Congestion map]({{ site.baseurl }}/assets/img/congestion-03.png)

![Congestion display modes]({{ site.baseurl }}/assets/img/congestion-04.png)

![Congestion view]({{ site.baseurl }}/assets/img/congestion-05.png)

Two things to know when reading it:

1. **Stippled versus solid nets.** After early global route, stippled nets are
   early-routed estimates; solid nets have been detail routed by NanoRoute.
2. **Diamond versus line display.** Global route congestion is shown as diamonds; NanoRoute
   congestion as horizontal and vertical lines. Either way, check whether an overflowing
   GCell is overflowing in the horizontal or the vertical direction — the fix differs.

![Global versus detail route congestion display]({{ site.baseurl }}/assets/img/congestion-07.png)

DRC violations cluster in congested areas, so the congestion map is usually also a map of
where your DRC problems will be.

### Fixing congestion

In rough order of cost:

1. **Cell padding** or **partial placement blockages** in the hot region, thinning the
   placement locally.
2. **Soft blockages in narrow channels**, if the congestion is structural.
3. **Re-place** the affected logic, or adjust placement effort.
4. **Move macros** or **expand the core** — back to the floorplan.

{: .interview }
> *"A channel between two macros is congested. Give three ways to fix it without changing
> the floorplan."* Thin the placement in and around the channel with cell padding or a
> partial blockage so fewer nets originate there. Add a soft blockage in the channel itself
> so cells are pushed out and the channel is left for through-routing. Reduce the routing
> demand crossing it — reorder scan chains that cross the channel unnecessarily, apply NDRs
> more selectively, or use path grouping to let optimisation restructure the logic that
> spans the two sides.

If congestion is high and **not localised to any particular area**, no local fix will help.
That is a floorplan problem: expand the design or change macro placement.

---

## Tracks and vias

### Tracks

Tracks are normally created automatically from the DEF and the technology LEF. `add_tracks`
is for when you need to override them.

![Adding tracks]({{ site.baseurl }}/assets/img/detail-route-01.png)

### Via selection

The router chooses vias automatically according to the rules in the LEF. The important
distinction is **single-cut** versus **multi-cut**.

![Via types]({{ site.baseurl }}/assets/img/detail-route-02.png)

A **multi-cut via** is several via cuts in parallel between the same two layers. It has
lower resistance and is far more robust against
[electromigration]({{ site.baseurl }}/docs/signoff/physical-verification/#em-electromigration),
because the current is spread across several paths and a single void does not open the
connection. As nodes shrink, via reliability becomes a bigger share of the problem, so
multi-cut vias on high-current nets are standard practice.

![Multi-cut vias]({{ site.baseurl }}/assets/img/detail-route-03.png)

{: .note }
> A sufficiently large single-cut via may be counted as multi-cut in reporting. Do not be
> surprised by the numbers.

### Via ladders, stacks, and pillars

All three connect a net across several metal layers, and they are closely related.

![Via stack]({{ site.baseurl }}/assets/img/via-ladders-via-stacks-and-via-pillars-01.png)

- **Via stack / via ladder** — vias stacked directly on top of one another to bring a net
  from a high layer down to a much lower one.
- **Via pillar** — a cross-parallel arrangement, structurally more like a lattice than a
  single column. It achieves the same layer traversal while providing multiple parallel
  current paths, which mitigates electromigration.

![Via pillar]({{ site.baseurl }}/assets/img/via-ladders-via-stacks-and-via-pillars-02.png)

`reportRouteTypeConstraints` reports how many via pillars were added.

---

## NDRs (non-default rules)
{: #ndrs-non-default-rules }

A **non-default rule** applies different routing geometry to selected nets: extra width,
extra spacing, or a restricted layer range.

![NDRs]({{ site.baseurl }}/assets/img/ndr-non-default-rules-01.png)

Candidates for NDRs:

- **Clock nets** — extra spacing reduces crosstalk-induced jitter; extra width reduces
  resistance and skew.
- **High-speed or critical signal nets** — where delay or noise margin is tight.
- **High-current nets** — wider metal for electromigration headroom.

NDRs are not free. A net with double width and double spacing consumes roughly four times
the routing resource of an ordinary one, so applying them liberally is a reliable way to
create congestion. Apply them to the
[top and trunk clock nets]({{ site.baseurl }}/docs/implementation/cts/#routing-the-clock-tree)
where the benefit is real, and leave the numerous leaf nets on default rules.

Attributes are set with `setAttribute` for individual nets, or `setRouteMode` for groups
that cannot be handled individually.

![setAttribute for NDRs]({{ site.baseurl }}/assets/img/route-02.png)

![Net attributes]({{ site.baseurl }}/assets/img/detail-route-08.png)

{: .note }
> Clock nets and critical nets are automatically given attributes granting them more
> spacing and shielding. Check what the tool has already done before adding your own.

### Shielding
{: #shielding }

**Shielding** routes grounded wires alongside a sensitive net, isolating it from
neighbouring switching signals.

![Shielding commands]({{ site.baseurl }}/assets/img/shielding-01.png)

![Shielding configuration]({{ site.baseurl }}/assets/img/shielding-02.png)

The shield wire absorbs the coupling that would otherwise reach the protected net, at the
cost of consuming adjacent tracks.

{: .tip }
> Shielding counts as a non-default rule and should be applied **before** ordinary routing.
> Retrofitting shields into an already-routed design means ripping up the neighbours.

---

## Yield and robustness optimisation

Post-route steps that improve manufacturability rather than timing.

**Wire spreading** redistributes spacing between nets so that wires are evenly separated
rather than clustered. This reduces coupling capacitance and cuts the amount of dummy metal
fill required later.

![Wire spreading]({{ site.baseurl }}/assets/img/detail-route-10.png)

**Post-route wire widening** widens wires where there is room. Wider wires have lower
resistance and better electromigration margin, and are less sensitive to manufacturing
variation.

![Wire widening]({{ site.baseurl }}/assets/img/detail-route-11.png)

**Patch wires** are small extra pieces of metal the detail router adds to fix **notch**
violations (a small concave step in a shape that the process cannot resolve) and
**minimum-area** violations (a shape too small to manufacture reliably). They are normal
output, not a sign of a problem.

---

## Configuring and reporting

![Detail route configuration]({{ site.baseurl }}/assets/img/detail-route-09.png)

| Command | Purpose |
|:--|:--|
| `report_route` | Routing statistics, including via types used |
| `reportWirePath -start <term> -end <term>` | Trace the actual path of a connection |
| `reportCongestion` | Congestion metrics |

**Antenna fixing modes** are configured here too. Layer hopping is the default method, and
the tool reads antenna rules and antenna cell names from the LEF.

![Antenna fixing modes]({{ site.baseurl }}/assets/img/detail-route-06.png)

![Antenna cell configuration]({{ site.baseurl }}/assets/img/detail-route-07.png)

---

## Debugging routing problems

### Common failure patterns

![Routing issue example]({{ site.baseurl }}/assets/img/detail-route-12.png)

![Routing issue example]({{ site.baseurl }}/assets/img/detail-route-13.png)

![Routing issue example]({{ site.baseurl }}/assets/img/detail-route-14.png)

![Routing issue example]({{ site.baseurl }}/assets/img/detail-route-15.png)

![Routing issue example]({{ site.baseurl }}/assets/img/detail-route-16.png)

### Fixing DRC violations

The blunt approach, which is often right:

```tcl
editDelete -regular_wire_with_drc
routeDesign -globalDetail
```

Delete everything with a violation and reroute it. The router with a cleaner starting point
frequently succeeds where incremental repair does not.

For manual work, the **Wire Editor** (press `e`) and its associated commands:

| Command | Purpose |
|:--|:--|
| `setEditMode` | Set wire editing properties |
| `editAddRoute` | Add a wire at given coordinates |
| `editSelect` / `editDeselect` | Select wires by region, property, or name |
| `editDelete` | Delete physical nets and their vias, or nets with DRC violations |
| `editCutWire` | Trim a wire |
| `editResize` | Change a wire's width or other physical properties |

![Manual route editing]({{ site.baseurl }}/assets/img/detail-route-17.png)

![Edit delete]({{ site.baseurl }}/assets/img/detail-route-18.png)

Use **Snap Settings** when manually routing to a pin, so the route attaches cleanly rather
than landing a fraction off-grid. The edit route panel can also create shielding.

![Route editing panel]({{ site.baseurl }}/assets/img/detail-route-19.png)

![Route repair]({{ site.baseurl }}/assets/img/detail-route-20.png)

![Route repair]({{ site.baseurl }}/assets/img/detail-route-21.png)

### When it is not a routing problem

Two cases where editing wires is wasted effort:

1. **Routing constrained around macro corners** — redo placement in that area.
2. **High global congestion, not localised** — go back to floorplanning, expand the design,
   or move macros.

### Examples of DRC violations

![DRC violation example]({{ site.baseurl }}/assets/img/detail-route-22.png)

![DRC violation example]({{ site.baseurl }}/assets/img/detail-route-23.png)

See [physical verification]({{ site.baseurl }}/docs/signoff/physical-verification/) for the
checks themselves.

---

## Sources

- Silicon Integration Initiative / Cadence, *LEF/DEF 5.8 Language Reference* — TRACKS, via
  rules, layer preferred direction, and spacing rules that constrain the router.
  <http://coriolis.lip6.fr/doc/lefdef/lefdefref/LEFSyntax.html>
- *Routing in VLSI Physical Design — Global, Detailed, DRC & SI*, EcrioniX — global versus
  detailed routing, antenna fixes by jumper and diode, and aggressor/victim coupling.
  <https://ecrionix.org/physical-design/routing/>
- *Concept of Wire Spreading*, EDABoard — wire spreading and its effect on required metal
  fill. <https://www.edaboard.com/threads/concept-of-wire-spreading.202043/>
- *What Are Multicut Vias — Advantage of It*, EDABoard — multi-cut vias for resistance and
  electromigration robustness.
  <https://www.edaboard.com/threads/what-are-multicut-vias-advantage-of-it.201441/>
- *Electromigration Concerns Grow In Advanced Packages*, Semiconductor Engineering — why
  shorter and wider conductors, and parallel current paths, extend mean time to failure.
  <https://semiengineering.com/electromigration-concerns-grow-in-advanced-packages/>
