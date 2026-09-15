---
layout: default
title: MOSFETs and CMOS Logic
parent: 1. Fundamentals
nav_order: 1
---

# MOSFETs and CMOS Logic
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## The transistor as a switch

Every digital chip you will ever work on is built from one component: the **MOSFET**
(metal-oxide-semiconductor field-effect transistor). For physical design purposes you can
treat it as a voltage-controlled switch with three terminals that matter — **gate**,
**source**, and **drain** — sitting on a shared **body** or substrate.

The voltage on the gate decides whether current can flow between source and drain. The
gate itself is separated from the channel by a very thin insulating layer of **gate
oxide**. Nothing flows *into* the gate; it only sets up an electric field. That single
fact drives most of what follows.

![MOSFET structure]({{ site.baseurl }}/assets/img/mosfets-01.png)

The **channel length** of a transistor is the separation between the source and drain
regions. This is the dimension that technology nodes are named after, and shrinking it is
what "moving to a smaller node" historically meant.

### NMOS and PMOS

There are two flavours, and they behave as opposites:

| | Gate at logic 0 | Gate at logic 1 | Good at passing |
|:--|:--|:--|:--|
| **NMOS** | OFF | ON | a strong 0 |
| **PMOS** | ON | OFF | a strong 1 |

An NMOS transistor conducts when its gate is driven high; a PMOS transistor conducts when
its gate is driven low. Neither one passes both logic levels well, which is exactly why
they are used in complementary pairs.

---

## CMOS: putting them together

**CMOS** stands for *complementary MOS*. A CMOS gate is built from two networks:

- a **pull-up network** of PMOS transistors connecting the output to VDD (supply), and
- a **pull-down network** of NMOS transistors connecting the output to VSS (ground).

The two networks are logical complements of each other. They are designed so that for any
valid input combination, exactly one of them is conducting. If the pull-up network is on,
the pull-down network is off, and vice versa.

### The inverter

The simplest possible CMOS gate. One PMOS on top, one NMOS below, gates tied together as
the input `A`, drains tied together as the output `Z`.

- When `A` is low: the NMOS is off, the PMOS is on, and `Z` is pulled up to VDD → output 1.
- When `A` is high: the PMOS is off, the NMOS is on, and `Z` is pulled down to VSS → output 0.

![CMOS inverter]({{ site.baseurl }}/assets/img/cmos-inverter-01.png)

### The NAND gate

Scale the same idea up. For a 2-input NAND with inputs `A` and `B`:

- **Pull-down network:** two NMOS transistors in *series*. The output is pulled to ground
  only when `A` **and** `B` are both high.
- **Pull-up network:** two PMOS transistors in *parallel*. The output is pulled to VDD
  when `A` is low **or** `B` is low — that is, `NOT A OR NOT B`.

![CMOS NAND]({{ site.baseurl }}/assets/img/cmos-nand-01.png)

Series NMOS in the pull-down, parallel PMOS in the pull-up, gives you NAND. Swap those
arrangements and you get NOR. This series/parallel duality is how every static CMOS gate
is constructed.

{: .tip }
> Notice that both of these gates are *inverting*. Static CMOS naturally builds inverting
> functions. A non-inverting AND is physically a NAND followed by an inverter, which is
> why AND cells are usually slower and larger than NAND cells of the same drive strength.
> This matters when the tool is
> [restructuring logic to close timing]({{ site.baseurl }}/docs/signoff/timing-closure/).

---

## Where delay and power actually come from

This is the section to remember. Three consequences of the CMOS structure explain most of
physical design.

### 1. Inputs are capacitors, not current sinks

A gate input connects to transistor gates, and gates are insulated by oxide. In steady
state, **no DC current flows into an input**. What an input presents to whatever drives it
is a **capacitive load** — the input capacitance of the next stage.

So when gate 1 drives gate 2, gate 1's output is not delivering current to a load in the
resistive sense. It is charging and discharging a capacitor.

### 2. Delay is charging time

Because switching means charging a capacitance through the finite resistance of a
conducting transistor, **delay rises with load capacitance**. A cell driving a large fanout
or a long wire takes longer to switch. This is the entire basis of

- why **buffers** are inserted on long nets (break one big capacitance into two smaller
  ones with a fresh driver between them),
- why **upsizing a cell** speeds it up (a wider transistor has lower on-resistance, so it
  charges the same capacitance faster),
- why upsizing has a **penalty** (a wider transistor also presents more input capacitance
  to *its* driver, pushing the problem one stage back), and
- why **wire parasitics** matter so much at advanced nodes. See
  [parasitic RC]({{ site.baseurl }}/docs/flow/design-inputs/#parasitic-rc-and-extraction).

### 3. Power splits into static and dynamic

When a CMOS gate is sitting at a stable logic 0 or logic 1, one network is off, so there
is no direct path from VDD to VSS. Ideally it draws no supply current at all. CMOS logic
therefore dissipates very little power while it is holding a value, and most of its power
is burned during transitions.

That is the idealised picture. Real transistors are imperfect switches, so a small
**leakage** current flows even in the off state, and at small geometries this static
component has grown to be a serious fraction of total power. The usual breakdown is:

| Component | When it happens | Covered in |
|:--|:--|:--|
| **Switching power** | Charging and discharging load capacitance | [Power analysis]({{ site.baseurl }}/docs/signoff/power-analysis/) |
| **Internal power** | Charging internal nodes inside the cell during a transition | [Power analysis]({{ site.baseurl }}/docs/signoff/power-analysis/) |
| **Short-circuit power** | Brief window during a transition when both networks conduct | [Power analysis]({{ site.baseurl }}/docs/signoff/power-analysis/) |
| **Leakage power** | Always, including when idle | [Power analysis]({{ site.baseurl }}/docs/signoff/power-analysis/) |

{: .warning }
> The short-circuit component is why **slow transitions are expensive**. A sluggish input
> edge keeps both networks partly on for longer, wasting power and degrading the output
> edge further down the path. This is why tools enforce a maximum transition constraint —
> see [transition time]({{ site.baseurl }}/docs/timing/sta-basics/#transition-time-and-slew).

---

## Threshold voltage flavours (VT)

Standard cell libraries usually ship the same logical cell in several **threshold voltage**
variants. The threshold voltage is roughly the gate voltage at which the transistor turns
on, and the foundry can tune it during manufacturing.

| Variant | Threshold | Speed | Leakage |
|:--|:--|:--|:--|
| **LVT** (low-VT) | Low | Fastest | Highest |
| **SVT** / **RVT** (standard/regular) | Medium | Medium | Medium |
| **HVT** (high-VT) | High | Slowest | Lowest |

Swapping a cell between VT flavours is one of the cheapest optimisation moves available to
a tool, because it does not change the cell's footprint — the layout is drop-in
compatible. Swap to LVT to recover timing on a critical path; swap to HVT on paths with
slack to recover leakage power.

{: .interview }
> *"When would you swap a low-VT cell for a high-VT cell?"* When the path has positive
> slack you do not need. You give up speed you were not using in exchange for lower
> leakage, with no area cost. The reverse swap — HVT to LVT — buys you speed on a critical
> path at the cost of leakage, and doing it indiscriminately across a whole block is the
> classic way to blow a leakage budget. See
> [optimising PPA]({{ site.baseurl }}/docs/signoff/timing-closure/#power-optimisation).

---

## Sources

- Silicon Integration Initiative / Cadence, *LEF/DEF Language Reference* — layer, via, and
  device terminology used throughout. <http://coriolis.lip6.fr/doc/lefdef/lefdefref/LEFSyntax.html>
- C. H. Kim, *CMOS Inverter: Power Dissipation and Sizing*, EE 5323 VLSI Design,
  University of Minnesota. <http://people.ece.umn.edu/~kia/Courses/EE5323/Slides/Lect_04_Inverter2.pdf>
- R. Amirtharajah, *CMOS Power Dissipation and Trends*, EEC 216, UC Davis — breakdown of
  dynamic, short-circuit, static, and leakage components.
  <https://www.ece.ucdavis.edu/~ramirtha/EEC216/W08/lecture1_updated.pdf>
- *Power Dissipation of a CMOS Inverter*, All About Circuits — on why "zero static power"
  is an idealisation and how leakage scales with node and temperature.
  <https://www.allaboutcircuits.com/technical-articles/power-dissipation-of-a-cmos-inverter/>
