---
layout: default
title: Clocks, Skew, and Latency
parent: 3. Timing
nav_order: 2
---

# Clocks, Skew, and Latency
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## The ideal clock tree
{: #the-ideal-clock-tree }

Before [clock tree synthesis]({{ site.baseurl }}/docs/implementation/cts/) runs, timing
tools model the clock as **ideal**. Specifically:

- The clock source has effectively **infinite drive** — it can supply whatever current
  every element in the clock network needs.
- Cells in the clock network are assumed to have **zero delay**.
- The clock therefore arrives at every sequential element **simultaneously**, with zero
  skew and no net delay.

None of this is true. It is assumed anyway, deliberately, so that early stages can
concentrate on **data paths** without every number being contaminated by a clock network
that does not physically exist yet.

Once CTS builds the real tree, the clock becomes **propagated** and its actual delays enter
the analysis. Numbers change at this point, and they generally change for the worse — which
is the reason the flow tries hard to have data path timing largely closed *before* CTS.

{: .warning }
> If you enter CTS with a large pile of pre-CTS violations, you are unlikely to close
> timing afterwards. Violation counts tend only to rise as the clock becomes real and
> parasitics become accurate. Fix what you can while the assumptions are still generous.

---

## Skew

**Skew**, in general, is the difference in arrival time between two or more signals. It
applies to data as well as clocks.

**Clock skew** specifically is the difference in clock arrival times at the endpoints
(sinks) of a clock tree — the clock pins of flip-flops.

![Clock skew]({{ site.baseurl }}/assets/img/clock-skew-01.png)

Skew is not automatically bad, and its effect depends on which check you are looking at:

| | Effect on setup | Effect on hold |
|:--|:--|:--|
| **Capture clock arrives late** (positive skew along the path) | Helps — more time available | Hurts — data races the late edge |
| **Capture clock arrives early** (negative skew) | Hurts — less time available | Helps |

This asymmetry is the whole basis of **useful skew** (below).

---

## Clock latency (insertion delay)
{: #clock-latency-insertion-delay }

**Clock latency** is the total time for the clock signal to travel from its source to an
endpoint. It is often split into:

- **Source latency** — from the true clock origin (a PLL or an external pin) to the clock
  definition point on the block boundary.
- **Network latency** (or **insertion delay**) — from that definition point through the
  clock tree to each sink.

In SDC:

```tcl
set_clock_latency 2.2 [get_clocks BZCLK]
# Applies to both rise and fall; use -rise / -fall if they differ.
# Use -source for source latency specifically.
```

Latency and skew are related but distinct: latency is *how long*, skew is *how uneven*. You
can have a clock tree with enormous latency and near-zero skew.

Large insertion delay is still undesirable, though, for a reason that is not obvious:

{: .interview }
> *"If your clock insertion delay is very high, how does that make OCV worse?"* Because
> on-chip variation is applied as a derate along the clock path. A longer path means more
> cells and more wire for the derate to act on, so the modelled divergence between the
> launch and capture clock paths grows. High insertion delay converts into large
> pessimism, which eats real timing margin even though nothing physically got slower. See
> [OCV]({{ site.baseurl }}/docs/timing/variation/#ocv-on-chip-variation).

---

## Clock uncertainty

You cannot directly tell a timing tool "assume 100 ps of skew". What you *can* do is
declare **clock uncertainty** — a margin subtracted from the available time, absorbing all
the effects that make a real clock edge arrive at an unpredictable moment.

```tcl
set_clock_uncertainty 0.250 -setup [get_clocks BZCLK]
set_clock_uncertainty 0.100 -hold  [get_clocks BZCLK]

set_clock_latency     2.0 [get_clocks USBCLK]
set_clock_uncertainty 0.2 [get_clocks USBCLK]
# e.g. 200 ps = 50 ps jitter + 100 ps skew + 50 ps extra pessimism
```

![Clock uncertainty]({{ site.baseurl }}/assets/img/clock-skew-02.png)

Uncertainty typically bundles:

- **Jitter** — cycle-to-cycle variation in the clock source itself. Every real oscillator
  has some.
- **Estimated skew** — before CTS, since the tree does not exist yet.
- **Margin** — deliberate extra pessimism the team wants to carry.

The practical effect is simple: uncertainty shrinks the effective time available for a
path. Adding 200 ps of setup uncertainty to a 2 ns clock is equivalent to validating the
same design at a higher frequency, since you are asking the same logic to complete in less
time.

Note the separate setup and hold values. They are usually different, and getting them
wrong is a classic source of confusing violations.

{: .tip }
> **Debugging tip.** If a large cluster of hold violations all sits in one clock domain or
> on one domain crossing, check the SDC before you touch the design. An excessive hold
> uncertainty on that domain — or a missing constraint on a crossing between two domains —
> will manufacture dozens of violations that no amount of buffer insertion deserves to fix.

After CTS, the estimated-skew portion of uncertainty is usually removed, because the tool
can now see the real skew.

---

## Useful skew and time borrowing

A perfectly balanced, zero-skew clock tree sounds like the goal. It is not, for three
reasons:

1. **Power and IR.** If every flop switches at exactly the same instant, the whole block
   draws its peak current simultaneously. That produces a large **IR drop** transient,
   which itself degrades timing — so enforcing zero skew can *cause* the problem it was
   supposed to avoid.
2. **Area and leakage.** Perfect balance is achieved by inserting a great many clock
   buffers. More cells means more area and more leakage power.
3. **Wasted opportunity.** Skew is a free resource for fixing timing, if you are willing to
   use it deliberately.

That third point is **useful skew**. Consider a path whose combinational delay is 5 ns in a
4 ns clock period. It violates setup by 1 ns. Rather than trying to make the logic 20%
faster, you can deliberately **delay the capture clock** by 1 ns. The path now has 5 ns
available and passes.

The time has to come from somewhere — you have borrowed it from the *next* stage, which now
has 3 ns instead of 4. This works when the next stage has slack to spare, which is often
the case.

The symmetric move also works: **remove buffers from the launch clock path** so data is
launched earlier and reaches the capture flop in time.

{: .warning }
> Useful skew moves a problem rather than deleting it. Borrow from a stage that has no
> slack and you have simply relocated the violation. CTS tools do this automatically and
> globally, which is why modern flows use **CCOpt**-style concurrent clock and datapath
> optimisation rather than building a balanced tree and then fixing timing separately.

See [CTS]({{ site.baseurl }}/docs/implementation/cts/) for how this is actually implemented.

---

## Lockup latches

A **lockup latch** is a level-sensitive latch inserted between two flip-flops to
deliberately add **half a clock cycle** of delay to a path.

![Lockup latch]({{ site.baseurl }}/assets/img/lockup-latches-01.png)

The main use is in [scan chains]({{ site.baseurl }}/docs/implementation/placement/#scan-chains).
Scan chains are long shift registers that snake through the design, and they are prone to
hold violations because:

- adjacent scan flops may be placed very close together, giving almost no combinational
  delay to work with, and
- a chain may cross between clock domains with different skew characteristics.

A latch that is transparent for half a cycle gives the data a guaranteed extra half period
before the next flop can capture it, which comfortably clears any hold requirement.

![Lockup latch in a scan chain]({{ site.baseurl }}/assets/img/lockup-latches-02.png)

You could get similar delay by inserting a long chain of buffers. The latch is usually
preferred because achieving half a cycle of delay in buffers costs substantial area, power,
and is sensitive to temperature and voltage — a latch gives you the delay by construction.

![Reordering scan chains with lockup latches]({{ site.baseurl }}/assets/img/lockup-latches-03.png)

---

## Sources

- X. Zhang, *Timing Analysis, Part 2*, ESE 461, Washington University in St. Louis — clock
  specification, source latency and network latency.
  <https://classes.engineering.wustl.edu/ese461/Lecture/week7b.pdf>
- R. Robucci, *Timing Analysis*, UMBC — on clock skew reducing the time available for a
  path to settle. <https://eclipse.umbc.edu/robucci/cmpeRSD/Lectures/Lecture13__TimingAnalysis/>
- *Lockup Latch*, SemiconShorts — why lockup latches are used on scan paths and clock
  domain crossings. <https://semiconshorts.com/2022/12/31/lockup-latch/>
- *Lockup Latches: Soul Mate of Scan Based Designs*, VLSI Universe — a longer treatment of
  lockup latch placement in scan chains.
  <https://vlsiuniverse.blogspot.com/2013/06/lockup-latches-soul-mate-of-scan-based.html>
