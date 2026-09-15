---
layout: default
title: The ASIC Design Flow
parent: 2. The Flow and Its Inputs
nav_order: 1
---

# The ASIC Design Flow
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## The whole picture

An ASIC goes from an idea to silicon through a long pipeline. Physical design sits near
the end, but it is easier to understand if you can see what came before and what comes
after.

![Full ASIC design flow]({{ site.baseurl }}/assets/img/design-flow-01.png)

| Stage | Input | Output | Question it answers |
|:--|:--|:--|:--|
| **Specification / architecture** | Requirements | Microarchitecture spec | What should the chip do, and how fast? |
| **RTL design** | Spec | Verilog/VHDL | How is the logic described? |
| **Functional verification** | RTL + testbench | Coverage, bug reports | Does the logic do the right thing? |
| **Synthesis** | RTL + `.lib` + SDC | Gate-level netlist | Which gates implement this logic? |
| **DFT insertion** | Netlist | Netlist + scan chains | How will we test the manufactured part? |
| **Physical design (P&R)** | Netlist + SDC + LEF + `.lib` + tech files | Routed layout | Where does everything physically go? |
| **Signoff** | Routed layout | Verified layout | Is it correct, fast enough, and manufacturable? |
| **Tapeout** | Verified layout | GDSII | Ship it to the foundry |

Everything from **floorplanning through routing** is what people mean by "physical design"
or "place and route" (P&R), and it is the subject of
[Part 4]({{ site.baseurl }}/docs/implementation/).

---

## The place-and-route flow in order

Within physical design, the steps run in a fixed order, because each one depends on
decisions made by the last.

1. **Design import / initialisation** — read the netlist, constraints, libraries, and
   technology files. Sanity-check that everything loaded.
2. **[Floorplanning]({{ site.baseurl }}/docs/implementation/floorplanning/)** — decide die
   and core size, place macros, define rows and blockages, assign I/O pins.
3. **[Power planning]({{ site.baseurl }}/docs/implementation/power-planning/)** — build the
   power distribution network: rings, stripes, rails.
4. **[Placement]({{ site.baseurl }}/docs/implementation/placement/)** — place the standard
   cells, interleaved with logic optimisation.
5. **[Clock tree synthesis (CTS)]({{ site.baseurl }}/docs/implementation/cts/)** — build and
   balance the physical clock network, then re-optimise.
6. **[Routing]({{ site.baseurl }}/docs/implementation/routing/)** — global route, then
   detail route, then post-route optimisation and repair.
7. **[Signoff]({{ site.baseurl }}/docs/signoff/)** — extraction, final STA, DRC, LVS,
   antenna, EM, IR drop, power. Fix by
   [ECO]({{ site.baseurl }}/docs/signoff/timing-closure/#eco-engineering-change-order).
8. **Export** — write out
   [GDSII]({{ site.baseurl }}/docs/signoff/physical-verification/#gdsii-and-tapeout).

{: .note }
> These stages are not strictly one-way. Discovering unfixable congestion during routing
> usually means going back to floorplanning. The further back you have to go, the more
> expensive the iteration — which is why floorplanning quality matters so
> disproportionately.

---

## The clock is ideal until it isn't

The single most important thing to understand about how the flow progresses is **when the
clock becomes real**.

**Before CTS**, the clock tree is treated as
[ideal]({{ site.baseurl }}/docs/timing/clocks/#the-ideal-clock-tree): the clock source is
assumed to have effectively infinite drive, clock network cells are assumed to have zero
delay, and the clock is assumed to arrive at every flip-flop at the same instant with zero
skew. This is obviously false, and deliberately so. It lets early stages focus entirely on
**data paths** without the clock network's delays muddying every number.

**After CTS**, the clock tree physically exists — real buffers, real wires, real
[insertion delay]({{ site.baseurl }}/docs/timing/clocks/#clock-latency-insertion-delay), real
[skew]({{ site.baseurl }}/docs/timing/clocks/#clock-skew). Timing analysis from that point
on uses the propagated clock, and the numbers change, sometimes dramatically.

This is why the same design gets timed repeatedly at different points in the flow, and why
a report is meaningless unless you know which stage produced it.

| Stage | Clock model | Wire delay model | What the numbers tell you |
|:--|:--|:--|:--|
| Pre-place | Ideal | Zero-wire-load / estimated | Are the constraints themselves sane? |
| Post-place | Ideal | Estimated from placement (virtual route) | Is the placement roughly workable? |
| Post-CTS | **Propagated** | Estimated | Does the real clock tree hold up? |
| Post-route | **Propagated** | **Extracted** from actual geometry | The real answer |

{: .tip }
> Running timing with the pre-route option zeroes out net capacitance so you see the most
> optimistic possible picture. If you already have violations *there*, you do not have a
> placement problem — you have a constraint problem. Either the design genuinely cannot
> make the frequency, or your SDC is missing
> [false paths and multicycle paths]({{ site.baseurl }}/docs/flow/design-inputs/#timing-exceptions).

---

## Signoff timing commands

The standard sequence for taking a design through to timing signoff:

![Signoff timing command flow]({{ site.baseurl }}/assets/img/design-flow-02.png)

Full command detail lives in the
[Innovus command reference]({{ site.baseurl }}/docs/tools/innovus-commands/).

---

## Sources

- R. Robucci, *Timing Analysis*, CMPE Reconfigurable System Design, UMBC — on static
  versus dynamic timing analysis and where each sits in the flow.
  <https://eclipse.umbc.edu/robucci/cmpeRSD/Lectures/Lecture13__TimingAnalysis/>
- X. Zhang, *Timing Analysis Part 2*, ESE 461, Washington University in St. Louis — clock
  specification, source latency, and network latency in STA.
  <https://classes.engineering.wustl.edu/ese461/Lecture/week7b.pdf>
- *LEF/DEF*, Wikipedia — on where LEF/DEF sits relative to place-and-route tools.
  <https://en.wikipedia.org/wiki/LEF/DEF>
