---
layout: default
title: Timing Closure, Optimisation, and ECO
parent: 5. Signoff
nav_order: 4
---

# Timing Closure, Optimisation, and ECO
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## Fixing timing violations

### General approaches, cheapest first

**Resize cells.** Upsize a cell on a critical path to drive its load faster. Cheap and local,
and the tool does it automatically — but it costs area and power, and it pushes more input
capacitance back onto the previous stage.

**Swap VT flavour.** Move a cell from HVT to SVT or LVT for speed, at the cost of leakage.
No area penalty, since the variants are footprint-compatible.

**Insert buffers.** Break a long or heavily-loaded net into segments with fresh drivers.

**Restructure the logic.** Replace a chain of simple gates with a complex cell that does the
same job in fewer stages — for instance replacing a NAND-plus-inverter arrangement with an
AOI (AND-OR-invert) cell.

**Use [useful skew]({{ site.baseurl }}/docs/implementation/cts/#useful-skew-and-time-borrowing).**
Borrow time from an adjacent stage with slack.

**Pipeline.** Insert intermediate registers to split a long combinational path across
multiple cycles. This changes the architecture and requires RTL cooperation, so it is a
last resort at the physical design stage — but it is the only real answer to a path that is
fundamentally too long.

**Reduce the clock frequency.** The honest admission that the design does not make the
target.

![Timing fix options]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-01.png)

### Check for `dont_touch` and `dont_use` first

Before concluding a path is hard, confirm the tool is actually allowed to fix it.

![Checking dont_touch]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-02.png)

![Attribute checks]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-03.png)

A `dont_touch` attribute prevents a cell from being modified at all, including resizing. A
misapplied one makes a perfectly ordinary path look immovable.

{: .tip }
> **A real debugging case.** After setup optimisation, WNS had not moved but TNS had
> improved. The cells on the critical path were placed close together, so it did not look
> like a placement problem — but inspecting them showed they were all the slowest,
> lowest-power variants. Something was stopping the tool upsizing them. The cause: a
> **`dont_use`** attribute had been applied to some cells where **`dont_touch`** was
> intended. The tool had been forbidden from using the faster variants at all.

### Fixing hold violations

![Hold fix tips]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-04.png)

Hold is fixed by **adding** delay — buffers, downsized cells, or
[lockup latches]({{ site.baseurl }}/docs/timing/clocks/#lockup-latches) on scan paths. Since
hold does not depend on the clock period, it cannot be escaped by slowing the clock.

{: .tip }
> **Check the constraints before the design.** If hold violations cluster in one clock
> domain, or all on paths crossing from one domain to another, investigate the SDC first.
>
> Two cases seen in practice: (1) the hold clock uncertainty on one domain was set far too
> high, manufacturing violations that did not exist; (2) a design had essentially no setup
> violations but dozens of large hold violations, all on paths crossing from `vclk2` to
> `vclk1` — there was no explicit constraint in the SDC for that crossing. The fix is a
> constraint, or lockup latches on those paths, not buffer insertion on each violation.

![Hold violation investigation]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-06.png)

![Clock domain crossing violations]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-07.png)

![Crossing analysis]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-08.png)

To get the top violating hold paths together with the tool's reasons for not fixing them:

![Top hold paths and reasons]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-09.png)

### When it is really a floorplan problem

![Floorplan-driven timing failure]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-05.png)

A floorplan with many narrow channels produces both congestion and broken timing paths,
because cells that should be near each other cannot be placed together. Adding a partial
blockage to the single worst area moved the congestion elsewhere; adding **soft blockages in
all the channels** was what worked. Symptoms are local; causes often are not.

### Fix violations before CTS

Violation counts tend only to rise as the flow progresses — the clock becomes real,
parasitics become accurate, and pessimism from
[OCV]({{ site.baseurl }}/docs/timing/variation/#ocv-on-chip-variation) gets applied. Entering
CTS with a large pile of pre-CTS violations rarely ends well.

---

## Which command optimises what
{: #which-command-optimises-what }

An easy thing to get wrong, because the defaults are not symmetric.

| Command | Optimises |
|:--|:--|
| `place_opt_design` | Setup only |
| `clock_opt_design` | Setup only (by default) |
| `clock_opt_design -cts` | Builds a balanced / zero-skew clock tree |
| `optDesign -postCTS` | Setup |
| `optDesign -postCTS -hold` | Hold |
| `route_opt_design` | Setup **and** hold |
| `optDesign -postRoute -setup -hold` | Both, after a `routeDesign` with no optimisation |
| `optDesign -postRoute -setup -hold -incr` | Both, incrementally |

`setOptMode -help` in the Innovus shell lists the full set of options.

The pattern: **hold fixing waits**. There is no point adding delay to fix hold before the
clock tree exists, because CTS will change the skew and therefore change how much delay you
actually need.

---

## Optimisation flow and reports

![Standard and express flows]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-10.png)

![Flow variants]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-11.png)

![Flow detail]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-12.png)

![Design reports]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-13.png)

![Report commands]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-14.png)

### Practical guidance for the whole flow

![Optimisation tips]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-15.png)

1. **Run `timeDesign -prePlace` before placement** to verify the constraints themselves.
   This is zero-wire-load timing — if it fails here, the problem is the SDC or the target
   frequency, not the implementation.
2. **Use the same constraints in implementation and signoff.** Optimising against one set of
   constraints and signing off against another is a reliable way to waste a week.
3. **Review the Early Global Route congestion report after optimisation**, not just the
   timing report.
4. **Leave 5–7% of your targeted final utilisation free** for optimisation to work in.
5. **After optimisation, examine the remaining violating paths individually.** Check that
   the worst path still meets your target slack. If not, use path groups to push harder on
   specific paths. Check the EGR routability numbers, and check the congestion map — if the
   worst paths run through local congestion hotspots, the timing problem is really a
   congestion problem.

### Signoff optimisation

![Signoff optimisation]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-16.png)

`flowEffort` and `powerEffort` can be set as appropriate for the signoff run.

To view timing and PPA metrics as HTML:

![HTML metrics]({{ site.baseurl }}/assets/img/methods-for-fixing-timing-issues-17.png)

---

## Path groups
{: #path-groups }

A **path group** buckets timing paths so the optimiser's cost function treats them
separately. Without grouping, one catastrophically failing path can dominate the cost
function and starve every other path of optimisation effort.

By default the tool creates two internal groups worth knowing:

- **`reg2reg`** — register to register, the core of the design.
- **`reg2clkgate`** — paths into clock gating enable pins, which are usually tight and worth
  watching separately.

![Path groups]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-04.png)

You can create additional groups and assign them higher optimisation priority to push on
difficult paths. The caveat: too many overlapping custom groups increases runtime
substantially, and the benefit falls off quickly.

---

## Optimising PPA

**PPA** — power, performance, area — is the three-way trade every implementation decision
sits inside.

![PPA]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-01.png)

![Optimisation transformations]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-02.png)

![Optimisation modes]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-03.png)

The command is available in the GUI under ECO → Optimize Design.

### Useful settings

**Extraction before detail route.** A different engine extracts RC after global routing but
before detail routing, to get realistic numbers from the planned routes:

```tcl
routeDesign -trackOpt
```

**Protecting critical paths from area reclamation.** Optimisation reclaims area and power by
downsizing cells with slack. Near critical paths this can be counterproductive — you want
some margin left. Set a small positive target slack below which area recovery will not be
attempted:

```tcl
setOptMode -opt_area_recovery_setup_target_slack <small positive number>
```

**Optimising TNS, not just WNS.** By default the engine targets the worst paths and is
relatively uninterested in the many mildly-violating ones. To make it work on all violating
endpoints:

```tcl
setOptMode -opt_all_end_points true
```

### Layer-aware buffering

![Layer-aware buffering]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-05.png)

Given a long net, the tool has two ways to fix its delay: insert a buffer, or route it on a
higher metal layer with lower resistance. If promoting the net to a higher layer achieves
the same delay, that is preferable — it avoids a cell entirely, saving power and area, and
higher layers are also better for signal integrity. The tool makes this trade automatically
as part of **area reclaim**.

![Adding repeaters]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-06.png)

### Power optimisation
{: #power-optimisation }

![Power-driven placement]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-07.png)

Enabling power-driven placement and optimisation changes the transformations the tool
prefers.

![Power-driven trade-offs]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-08.png)

The trade is subtle. **Simple transformations** — swapping a low-drive cell for a
high-drive one — close timing but increase power. **More complex transformations** —
restructuring logic, rebuffering — can close timing *and* reduce power, and become worth
their runtime when timing is hard to close.

**Leakage-to-dynamic ratio.** You can tell the tool how to weight leakage against dynamic
power, which changes which optimisations it prefers:

![Leakage to dynamic ratio]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-09.png)

![Ratio results]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-10.png)

**Global optimisation versus blanket swapping.** This is the key point:

![Power optimisation techniques]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-11.png)

A naive approach swaps **all** HVT cells for SVT and LVT to gain speed. The result is
enormous leakage for very little benefit, because most of those paths had slack and did not
need the speed. **Global optimisation** instead identifies the paths that actually violate
and swaps only the cells on those bottlenecks.

![Timing and power trade-off]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-12.png)

![Meeting timing just barely]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-13.png)

The underlying principle: **timing and power are inversely related, and timing should be met
just barely**. Every picosecond of slack beyond your target was bought with power you did
not need to spend. A design that closes at +200 ps of WNS is a design that is burning power
for nothing.

### Power commands

![Power optimisation settings]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-14.png)

![Power results]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-15.png)

| Command | Purpose |
|:--|:--|
| `read_activity_file` | Read a switching activity file, e.g. SAIF |
| `set_default_switching_activity` | Activity for nets not explicitly annotated |
| `report_power` | Power report; can include clock pin power |
| `all_fanin` / `all_fanout` | Report the fanin/fanout of a logic structure |

![report_power]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-16.png)

![all_fanin and all_fanout]({{ site.baseurl }}/assets/img/methods-for-optimizing-ppa-17.png)

{: .interview }
> *"You have a path violating setup by −50 ps and no space to upsize cells. What do you
> do?"* Look beyond sizing. Use useful skew to borrow time from an adjacent stage. Swap the
> path's cells to a lower VT rather than a larger footprint — same area, more speed. Check
> whether the path is genuinely constrained, or whether it should be a multicycle or false
> path. Reroute it on a higher metal layer, or apply an NDR to reduce its resistance. Move
> the endpoints closer via a placement constraint. Restructure the logic into fewer stages.
> If none of that works, the honest answer is that the floorplan needs to change or the
> frequency target does.

---

## ECO (Engineering Change Order)
{: #eco-engineering-change-order }

An **ECO** is a targeted change to an otherwise-finished design. The defining constraint is
minimality: you want to fix the problem while disturbing as little as possible of a design
that has already passed everything else.

### Premask versus postmask

**Premask ECO** — no masks have been committed yet, so any layer may change. You have full
freedom: add cells, remove cells, re-place, reroute.

**Postmask ECO** — the base layers (transistors) have already been manufactured. Only the
**metal** layers can change. This means no new transistors: any new logic must come from
[spare cells]({{ site.baseurl }}/docs/fundamentals/cells/#spare-cells) already sitting in the
design, rewired through metal-only changes.

`ecoDesign` can be restricted to operate only on specified layers, which is how you prevent
a postmask ECO from touching something already taped out.

![Premask versus postmask ECO]({{ site.baseurl }}/assets/img/eco-engineering-change-order-07.png)

`ecoPlace` behaves differently depending on which you are doing:

| Mode | `ecoPlace` behaviour |
|:--|:--|
| **Postmask** | Maps unplaced cells onto existing placed spare cells |
| **Premask** | Moves unplaced cells into the core area normally |

![ecoPlace]({{ site.baseurl }}/assets/img/eco-engineering-change-order-08.png)

![ecoPlace modes]({{ site.baseurl }}/assets/img/eco-engineering-change-order-09.png)

### Interactive ECO

ECO → Interactive ECO lets you manually add, delete, or swap an instance and immediately see
the effect on the design.

![setEcoMode]({{ site.baseurl }}/assets/img/eco-engineering-change-order-01.png)

`setEcoMode` configures what types of ECO change are permitted.

| Command | Purpose |
|:--|:--|
| `ecoAddRepeater` | Insert a buffer or inverter |
| `ecoChangeCell` | Swap one cell for another |
| `attachTerm` | Connect a cell to another cell's terminal |
| `detachTerm` | Disconnect it |

![ecoAddRepeater]({{ site.baseurl }}/assets/img/eco-engineering-change-order-02.png)

![ecoChangeCell]({{ site.baseurl }}/assets/img/eco-engineering-change-order-03.png)

![attachTerm]({{ site.baseurl }}/assets/img/eco-engineering-change-order-04.png)

![detachTerm]({{ site.baseurl }}/assets/img/eco-engineering-change-order-05.png)

### Scripted ECO

Rather than reading in a whole replacement netlist, `loadECO` reads a file of ECO
**directives** describing only the changes to be made. The file is **not** Tcl, and all
directives are written in capitals.

![loadECO]({{ site.baseurl }}/assets/img/eco-engineering-change-order-06.png)

This is the normal way large ECOs are applied, because it makes the change reviewable and
repeatable — you can diff the directive file, which you cannot meaningfully do with two
netlists.

### Parallel and spare flows

![Parallel ECO flow]({{ site.baseurl }}/assets/img/eco-engineering-change-order-10.png)

![Spare cell flow]({{ site.baseurl }}/assets/img/eco-engineering-change-order-11.png)

### Signoff commands

![ECO signoff]({{ site.baseurl }}/assets/img/eco-engineering-change-order-12.png)

{: .warning }
> After any ECO, rerun [LVS]({{ site.baseurl }}/docs/signoff/physical-verification/#lvs-layout-versus-schematic).
> Manual ECO edits are the single most common source of LVS mismatches, and an ECO that
> fixes timing while breaking connectivity is worse than no ECO at all.

---

## Sources

- *E2ESlack*, arXiv — slack as the metric used to pinpoint paths for optimisation, and the
  role of pre-routing prediction in placement optimisation. <https://arxiv.org/pdf/2501.07564>
- R. Robucci, *Timing Analysis*, UMBC — on pessimistic worst-case delay assumptions in STA
  driving synthesis and optimisation choices.
  <https://eclipse.umbc.edu/robucci/cmpeRSD/Lectures/Lecture13__TimingAnalysis/>
- A. Mishra, *Static Timing Analysis: Setup and Hold Time* — standard remedies for failing
  paths: timing exceptions, pipelining long combinational paths, and adjusting clocking.
  <https://medium.com/@hi.avimishra2/static-timing-analysis-d3fdc2c42a70>
- *Power Dissipation of a CMOS Inverter*, All About Circuits — the leakage cost of
  lower-threshold devices, underlying the VT swap trade-off.
  <https://www.allaboutcircuits.com/technical-articles/power-dissipation-of-a-cmos-inverter/>
- *Clock Tree Optimization Methodologies*, Semiconductor Digest — clock gating as the primary
  clock power saving, and its timing implications.
  <https://www.semiconductor-digest.com/clock-tree-optimization-methodologies-for-power-and-latency-reduction/>
