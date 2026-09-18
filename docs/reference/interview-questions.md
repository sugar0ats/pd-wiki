---
layout: default
title: Interview Questions
parent: 7. Study and Reference
nav_order: 2
---

# Interview Questions
{: .no_toc }

Physical design interview questions, grouped by topic, each linked to the page that covers
the underlying material.
{: .fs-5 .fw-300 }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## How these interviews tend to go

The pattern reported from physical design interviews at large companies is a focus on **the
physics and the trade-offs behind each decision**, rather than on tool command syntax. The
questions are usually of the form *why does the tool do X* and *what does X cost you*, and
the follow-up is usually *and what breaks when you do that*.

Practical implications:

- Know **why** each flow step exists and what goes wrong without it.
- Be able to state the **penalty** for any fix you propose. Every fix costs something.
- Be ready to **debug** when the standard answer does not work. "Upsize the cell" is a
  first move, not an answer.
- Know what **RC delay, slew, and capacitance** actually are, not just that they matter.

---

## Fundamentals and trade-offs

**You have a setup violation. You upsize the cell to fix it. What's the penalty?**
More area, more leakage and switching power, and more input capacitance presented to the
previous stage — so you may have moved the critical path one stage back. You may also have
created a hold violation on the same path, and you have consumed placement space that
optimisation later wanted.
→ [MOSFETs and CMOS]({{ site.baseurl }}/docs/fundamentals/mosfets-and-cmos/#where-delay-and-power-actually-come-from)

**When would you swap a low-VT cell for a high-VT cell? How does that affect timing versus
power?**
When the path has slack you do not need. HVT is slower and leaks far less, with no area
penalty since the variants are footprint-compatible. The reverse swap buys speed on a
critical path at a leakage cost — and doing it indiscriminately across a block is how
leakage budgets get blown.
→ [Threshold voltage flavours]({{ site.baseurl }}/docs/fundamentals/mosfets-and-cmos/#threshold-voltage-flavours-vt)

**Why does wire delay dominate gate delay at 7 nm and 5 nm?**
Transistors scaled better than interconnect did. Wires got thinner and narrower, so
resistance per unit length rose sharply, while they also got closer together, so coupling
capacitance rose. Gate delays fell with each node; wire RC did not follow. At advanced nodes
a net's RC can exceed the delay of the cell driving it, which is why placement quality and
layer assignment matter so much more than they used to.
→ [Parasitic RC]({{ site.baseurl }}/docs/flow/design-inputs/#parasitic-rc-and-extraction)

**If you fix a setup violation and it creates a hold violation on the same path, what caused
that?**
Setup and hold constrain the same path from opposite ends. Making the path faster —
upsizing, VT swap, rerouting — moves it from "too slow to arrive in time" towards "fast
enough to race through and corrupt the capture". Any fix that moves delay in one direction
eats margin in the other.
→ [Setup and hold]({{ site.baseurl }}/docs/timing/sta-basics/#setup-and-hold)

---

## Floorplanning and congestion

**Why place macros on the periphery instead of the centre? What happens to routability?**
Central macros fragment the placeable area and force every crossing net to detour, raising
wire length and concentrating congestion in the channels around them. Peripheral macros
leave one contiguous core region. Secondary effect: a central macro blocks power stripes
from crossing, so the region behind it is furthest from any supply and droops most.
→ [Macro placement]({{ site.baseurl }}/docs/implementation/floorplanning/#macro-placement)

**How does your floorplan affect IR drop? Where would you place power straps for a
high-density logic block?**
Macros block stripes, so regions shadowed by macros — particularly between two of them — sit
furthest from the supply and have the highest series resistance. For a high-density block
you want stripes running directly over it at tighter pitch, with a solid via stack down to
the rails, and you want it not boxed in by macros on several sides.
→ [Power planning]({{ site.baseurl }}/docs/implementation/power-planning/)

**A channel between two macros is congested. Give three ways to fix it without changing the
floorplan.**
Thin the placement in and around the channel with cell padding or a partial blockage. Add a
soft blockage in the channel so cells are pushed out and the space is left for through
routing. Reduce demand crossing it — reorder scan chains that cross unnecessarily, apply
NDRs more selectively, or use path groups to let optimisation restructure the logic spanning
the two sides.
→ [Congestion]({{ site.baseurl }}/docs/implementation/routing/#congestion)

**Why does pin density matter more than cell density in advanced nodes?**
Cells shrank faster than the lowest metal layers' pitch did. A legal, comfortably-spaced
placement can still be unroutable because there are not enough M1/M2 tracks to reach every
pin in that neighbourhood. You can have modest cell density and a region that cannot be
routed.
→ [Routing tracks]({{ site.baseurl }}/docs/fundamentals/cells/#routing-tracks)

**Draw a floorplan for a design with five macros and explain your reasoning.**
Talk through it rather than drawing silently. Macros on the periphery; pins oriented towards
the logic that uses them; macros that communicate placed adjacent; no narrow channels left
between them — either widen or close them; halos around each; power stripes planned so no
logic region is shadowed by two macros at once; a contiguous standard cell region left in
the middle.
→ [Floorplanning]({{ site.baseurl }}/docs/implementation/floorplanning/)

---

## CTS and timing closure

**What's the difference between an H-tree and a mesh clock network? Which uses more power?**
The mesh. A mesh is a shorted grid driven from many points: excellent skew and OCV
tolerance, but highly capacitive, resource-hungry, and it cannot be gated, so it dissipates
continuously. An H-tree is a tree, so it can be gated and uses less power, and achieves low
skew via matched branch lengths — but it wants regular unblocked area that a macro-heavy
floorplan will not provide.
→ [Clock tree architectures]({{ site.baseurl }}/docs/implementation/cts/#clock-tree-architectures)

**Is zero skew actually the goal in CTS? Why or why not?**
No. Simultaneous switching produces a large IR droop that degrades timing across the block;
perfect balance costs a great many buffers in area and leakage; and skew is a usable resource
for fixing timing. Modern flows deliberately use skew rather than eliminating it.
→ [Why zero skew is not the goal]({{ site.baseurl }}/docs/implementation/cts/#why-zero-skew-is-not-the-goal)

**If your clock insertion delay is very high, how does that make OCV worse?**
OCV derating is applied along the clock path. A longer path has more cells and more wire for
the derate to act on, so the modelled divergence between launch and capture clock paths
grows. High insertion delay converts directly into pessimism that eats real margin.
→ [Clock latency]({{ site.baseurl }}/docs/timing/clocks/#clock-latency-insertion-delay)

**How does clock gating save power? Can it cause timing violations?**
It stops the clock toggling to registers doing no useful work, removing both their switching
power and the clock network power below the gate — the largest dynamic power saving
available. It can cause violations: the enable into the gate has its own setup and hold
requirement against the clock, and it is usually tight. Tools create a `reg2clkgate` path
group precisely because these paths need watching.
→ [CTS]({{ site.baseurl }}/docs/implementation/cts/)

**You have a path violating setup by −50 ps and no space to upsize cells. What do you do?**
Useful skew to borrow time from an adjacent stage. VT swap rather than resize — same area,
more speed. Check whether the path should be a multicycle or false path at all. Reroute on a
higher metal layer or apply an NDR to cut its resistance. Pull the endpoints closer with a
placement constraint. Restructure the logic into fewer stages. If none of that works, the
floorplan or the frequency target has to change.
→ [Timing closure]({{ site.baseurl }}/docs/signoff/timing-closure/)

---

## Practical debugging

**A net is failing due to crosstalk noise. You can't move the net. How do you fix it?**
Work on the endpoints and the aggressor instead of the geometry. Upsize the victim's driver
so it holds the net harder against injected charge. Downsize or slow the aggressor if it has
slack. Buffer the victim to shorten the coupled segment. Shield it, which changes the
neighbours rather than the net. Swap the receiving cell for one with a better noise margin.
→ [Signal integrity]({{ site.baseurl }}/docs/signoff/signal-integrity/#fixing-si-problems)

**What happens if your `.lib` file doesn't match your operating voltage?**
Every delay number is wrong. Liberty delay tables are characterised at a specific voltage;
analysing at a different one means your timing describes a chip that does not exist. This is
exactly the trap when rail analysis shows cells running below nominal but timing is run at
nominal. Either carry a library characterised at the real voltage or budget the gap
explicitly in uncertainty.
→ [Power and rail analysis]({{ site.baseurl }}/docs/signoff/power-analysis/#rail-analysis)

**What's the difference between a false path and a multicycle path in SDC?**
A false path is never functionally exercised and is removed from analysis entirely. A
multicycle path is genuinely exercised but is allowed N cycles rather than one, so it is
still checked — just against a different edge. Declaring something false when it is really
multicycle means you have stopped checking a path that can actually fail.
→ [Timing exceptions]({{ site.baseurl }}/docs/flow/design-inputs/#timing-exceptions)

**What does an LVS check look for? What happens if it fails?**
It extracts a netlist from the layout and compares it device by device and net by net against
the source netlist. A mismatch means the silicon would not implement the verified design, so
it is a hard tapeout blocker regardless of timing quality. Usual causes: ECO mistakes, power
connection errors, shorts introduced by manual routing edits.
→ [LVS]({{ site.baseurl }}/docs/signoff/physical-verification/#lvs-layout-versus-schematic)

---

## Questions worth being able to answer

Not from any specific interview, but each one exposes whether you understand a mechanism
rather than a rule:

- Why is a setup violation more serious than a hold violation?
- Why is the clock tree built *after* placement rather than before?
- What is the difference between a corner and a mode, and why do you need several of each?
- Why does GBA report violations that PBA does not?
- Why do long nets cause antenna violations, and why does jumping to a higher layer fix it?
- What is a via pillar for, and how is it different from a via stack?
- Why do you leave 5–7% of utilisation free going into placement?
- What is in a LEF file that is not in a Liberty file, and vice versa?
- What is a spare cell for, and when can you no longer use one?
- Why does `place_opt_design` optimise setup but not hold?

---

## Also expect

A general discussion of academic projects, internship experience, familiarity with PD tools
(Cadence Innovus, Synopsys ICC2), and the complete flow from netlist to GDSII. Being able to
narrate that flow start to finish, naming what each stage consumes and produces, is worth
practising out loud.

---

## Sources

- Question sets in the first four sections are as circulated publicly on LinkedIn from a
  reported Physical Design Engineer interview at Google India. Answers here are written
  against the material in this wiki and its cited sources, not supplied with the questions.
- *Ultimate Guide: Clock Tree Synthesis*, AnySilicon. <https://anysilicon.com/clock-tree-synthesis/>
- *Clock Tree Optimization Methodologies for Power and Latency Reduction*, Semiconductor
  Digest. <https://www.semiconductor-digest.com/clock-tree-optimization-methodologies-for-power-and-latency-reduction/>
- *Routing in VLSI Physical Design*, EcrioniX. <https://ecrionix.org/physical-design/routing/>
- *Antenna Effect in 16nm Technology Node*, Design & Reuse.
  <https://www.design-reuse.com/article/61201-antenna-effect-in-16nm-technology-node-/>
