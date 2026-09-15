---
layout: default
title: Variation, Corners, and Analysis Modes
parent: 3. Timing
nav_order: 3
---

# Variation, Corners, and Analysis Modes
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## The problem: one design, many realities

A single netlist will be manufactured millions of times, across process variation, and each
copy will run at varying temperature and supply voltage, in several operating modes. The
timing has to work for all of it.

STA handles this by analysing the design repeatedly under different sets of assumptions.
The vocabulary for those assumptions is **corners**, **modes**, and **views**.

---

## Corners, modes, and MMMC
{: #corners-modes-and-mmmc }

The two concepts are easy to confuse, so pin them down:

**Corner** — the *physical and environmental* conditions. Process skew (fast/typical/slow),
supply voltage, temperature. These are the PVT conditions the library was characterised at,
written like `125c_0p8v`. A corner also carries the RC extraction settings appropriate to
those conditions.

**Mode** — the *functional state* the design is operating in. Functional mode, scan shift
mode, test mode, low-power mode, and so on. Different modes have different active clocks
and different constraints, so each mode has its own SDC.

A **view** is a pairing of one mode with one corner. **MMMC** — multi-mode multi-corner —
is analysing across many such pairings at once.

![MMMC concept]({{ site.baseurl }}/assets/img/mmmc-01.png)

### Why you need several

Setup and hold want opposite extremes:

| Check | Wants the design to be | Library corner |
|:--|:--|:--|
| **Setup** | As slow as possible | Slow / worst-case (SS, high temp, low voltage) |
| **Hold** | As fast as possible | Fast / best-case (FF, low temp, high voltage) |

At absolute minimum you configure **two library sets** — one worst-case and one best-case —
and analyse setup on one, hold on the other. Real production flows carry many more.

### Building an MMMC view definition

The pieces stack up in a specific order:

1. **Library set** — a named bundle of `.lib` files (plus, optionally, their associated
   files) that can be reused. `create_library_set`.
2. **RC corner** — the extraction conditions, including the extraction tech file and
   temperature. `create_rc_corner`.
3. **Delay corner** — combines a library set with an RC corner. `create_delay_corner`. If
   you lack proper library data you can substitute a **virtual operating condition**
   instead.
4. **Constraint mode** — a named set of SDC files. `create_constraint_mode`.
5. **Analysis view** — pairs one constraint mode with one delay corner.
   `create_analysis_view`.
6. **Active views** — `set_analysis_view` declares which views are actually used, and
   separately for setup and for hold.

![MMMC view file example]({{ site.baseurl }}/assets/img/mmmc-02.png)

![MMMC flow diagram]({{ site.baseurl }}/assets/img/mmmc-03.png)

![MMMC setup commands]({{ site.baseurl }}/assets/img/mmmc-04.png)

![Library set and RC corner creation]({{ site.baseurl }}/assets/img/mmmc-05.png)

The GUI setup wizard takes the collateral (`.sdc`, `.lib`, extraction files) and emits the
view definition file; the commands above are what it runs underneath. Writing the file
directly is normal once you know the flow.

### Reporting across views

`timeDesign` reports results per active view, so you can see at a glance whether a
violation is specific to one corner or present everywhere.

![timeDesign across views]({{ site.baseurl }}/assets/img/mmmc-06.png)

{: .tip }
> An **incomplete MMMC setup does not produce an error** — it produces confident, wrong
> numbers. If you omit a corner, the tool simply never checks it, and a violation that only
> appears at, say, low temperature will sail through implementation and surface at signoff.
> Under-constraining is the more dangerous failure mode because it looks like success.

---

## OCV (On-Chip Variation)
{: #ocv-on-chip-variation }

Corners capture variation *between* chips. **On-chip variation** captures variation *within
a single die*.

Two instances of the same cell, on the same die, do not behave identically. One region may
be hotter than another; the supply may droop more in one place; lithography and doping vary
slightly across the wafer. So a cell in the corner of the block may switch measurably
slower than an identical cell in the middle.

![OCV concept]({{ site.baseurl }}/assets/img/ocv-on-chip-variation-01.png)

### Derating

The classic way to model this is **derating** — applying a multiplier to the delays of
early and late paths to add a safety margin. The direction depends on which check you are
protecting:

**For setup**, you want the worst case for data arriving in time. So you make the **launch
path slow** and the **capture clock path fast** — the capture edge arrives sooner, leaving
less time.

![Setup derating]({{ site.baseurl }}/assets/img/ocv-on-chip-variation-02.png)

**For hold**, the opposite. You make the launch path fast and the **capture clock path
slow**, maximising the chance of a race.

![Hold derating]({{ site.baseurl }}/assets/img/ocv-on-chip-variation-03.png)

### AOCV (Advanced OCV)

Flat derate factors are blunt instruments. Applying, say, ±5% to every cell on every path
is wildly pessimistic for long paths, because random variation along a long path tends to
**average out** — some cells are faster than nominal, some slower, and they partially
cancel. A short path has no such averaging, so its variation really can be extreme.

**Advanced OCV** uses this. Instead of one factor, it derives derate factors from the
path's **depth** (and often its physical **distance**): deeper paths get smaller derates,
shallow paths get larger ones.

![AOCV depth-based derating]({{ site.baseurl }}/assets/img/ocv-on-chip-variation-04.png)

![AOCV tables]({{ site.baseurl }}/assets/img/ocv-on-chip-variation-05.png)

The payoff is recovering margin that flat OCV was throwing away. At advanced nodes this
matters enough that further refinements exist — **POCV/SOCV** (parametric/statistical OCV),
which model each cell's delay as a statistical distribution rather than a fixed derate.

{: .note }
> Notice the recurring theme: STA is pessimistic by construction, and every generation of
> analysis technique is an attempt to remove pessimism that is not physically realisable —
> without removing pessimism that is.

---

## PBA versus GBA

Two ways of computing timing on a path, trading accuracy against runtime.

**GBA (graph-based analysis)** propagates a single worst-case value through the timing
graph. At each node it keeps the worst slew and worst arrival time seen from any incoming
arc, and carries that forward.

**PBA (path-based analysis)** re-times each path individually, propagating the slew that
*actually* occurs along that specific path.

![PBA versus GBA]({{ site.baseurl }}/assets/img/pba-vs-gba-01.png)

| | GBA | PBA |
|:--|:--|:--|
| Slew used | Worst slew at each node, regardless of path | The actual slew along this path |
| Runtime | Fast | Slow |
| Pessimism | High | Low |
| Typical use | Everywhere, all the time | Final signoff, on the worst paths only |

The pessimism is real. A node might see its worst slew from a path that is not the path
under analysis, and GBA will still apply that slew — modelling a combination of conditions
that cannot actually co-occur.

Practical flows use GBA throughout and then run PBA on the remaining violating paths at
signoff. It is common for a chunk of apparent violations to simply disappear under PBA,
because they were pessimism rather than genuine failures.

---

## Running timing analysis

Some practical mechanics, in Innovus terms.

**Pre-route timing.** Running with the pre-route option zeroes out net capacitance,
producing the most optimistic possible result. This is a **constraint sanity check**, not a
timing check: if the design violates here, either the design genuinely cannot meet the
frequency, or the SDC is missing
[exceptions]({{ site.baseurl }}/docs/flow/design-inputs/#timing-exceptions).

![Pre-route timing]({{ site.baseurl }}/assets/img/performing-timing-analysis-01.png)

**Analysing a single net or cell.**

![Single net analysis]({{ site.baseurl }}/assets/img/performing-timing-analysis-02.png)

**Analysing specific constraint types** — reporting by endpoint, filtered to setup or hold:

![Constraint-specific reports]({{ site.baseurl }}/assets/img/performing-timing-analysis-03.png)

**Formatting reports.** The `-format` flag rearranges which metrics appear as columns,
which matters more than it sounds when you are scanning hundreds of paths.

![Report formatting]({{ site.baseurl }}/assets/img/performing-timing-analysis-04.png)

**Path groups** let you bucket paths for separate analysis and optimisation:

![Path group commands]({{ site.baseurl }}/assets/img/performing-timing-analysis-05.png)

![Custom path groups]({{ site.baseurl }}/assets/img/performing-timing-analysis-06.png)

**In the GUI**, the timing analysis windows let you cross-probe from a failing path to its
physical location, which is usually how you discover that a violation is really a
[congestion]({{ site.baseurl }}/docs/implementation/routing/#congestion) or floorplan
problem.

![GUI timing analysis]({{ site.baseurl }}/assets/img/performing-timing-analysis-07.png)

Delay calculation itself is configured globally with `setDelayCalMode`, which sets accuracy
level, corner-based analysis options, and handling of special cases such as high fanout
nets and input slew sensitivity. `getDelayCalMode` reports the current settings.

---

## Sources

- *Methods and Apparatus for Performing Statistical Static Timing Analysis*, US Patent
  8,645,881 — early/late path delay decomposition and the role of derating and adjustment
  terms in slack. <https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/8645881>
- *Method... for Performing a Parameterized Statistical Static Timing Analysis*, US Patent
  8,468,483 — setup and hold margin interdependence and timing yield.
  <https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/8468483>
- *E2ESlack*, arXiv — on post-routing STA being performed across early/late and rise/fall
  corner conditions. <https://arxiv.org/pdf/2501.07564>
- *MMMC File Setup for PnR Using Innovus*, Digital System Design — library set, RC corner,
  delay corner, constraint mode, and analysis view creation order.
  <https://digitalsystemdesign.in/mmmc-file-setup-for-pnr-using-innovus/>
