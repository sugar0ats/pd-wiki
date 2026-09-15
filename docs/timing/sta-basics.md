---
layout: default
title: STA Basics
parent: 3. Timing
nav_order: 1
---

# STA Basics
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## What "static" means

**Static timing analysis (STA)** checks whether a design meets its timing requirements
*without simulating it*. It uses no test vectors and no input data at all. Instead it
treats the design as a graph of timing arcs, walks every path structurally, and computes
the worst delay through each one.

That is the contrast with **dynamic timing analysis**, which simulates the circuit with
actual stimulus and examines the resulting waveforms. Dynamic analysis tells you what
happens for the vectors you chose; it cannot tell you about the path you forgot to
exercise. Static analysis is exhaustive over paths and dramatically faster, which is why it
is the signoff method for synchronous digital design.

The tradeoff is pessimism. Because STA does not know what data is on a path, it assumes the
worst plausible combination of conditions everywhere. Much of advanced timing analysis
(see [variation]({{ site.baseurl }}/docs/timing/variation/)) is about clawing back pessimism
that is not physically achievable.

{: .note }
> STA assumes a **synchronous** design. Asynchronous logic, combinational loops, and clock
> domain crossings all need special handling — either constraints telling STA to ignore
> them, or explicit synchroniser structures.

### When STA runs

First after synthesis, once there is a gate-level netlist to analyse. Then repeatedly
throughout physical design, with progressively more realistic assumptions, and finally at
signoff with extracted parasitics and a propagated clock.

The goal at every point is the same: **find the worst setup and hold paths**, and
understand why they are worst.

---

## Setup and hold

A flip-flop does not capture data instantaneously. It has a small window around the active
clock edge during which its data input must be stable. Violate either side of that window
and the flop may fail to capture a clean value.

**Setup time** is the interval *before* the clock edge during which data must already be
stable.

> A **setup check** ensures data arrives at the capture flip-flop early enough — that the
> combinational path from launch to capture fits within the clock period, minus the setup
> requirement.

**Hold time** is the interval *after* the clock edge during which data must remain stable.

> A **hold check** ensures data does not change too soon — that new data launched by the
> same edge does not race through the combinational logic and overwrite the value the
> capture flop is still latching.

The two checks pull in opposite directions:

| | Setup | Hold |
|:--|:--|:--|
| Concerned with | The **slowest** path | The **fastest** path |
| Analysed using | **Slow** library corner | **Fast** library corner |
| Fixed by | Making the path faster | Making the path slower |
| Depends on clock period? | Yes | **No** |

That last row surprises people. Hold is a race between two paths launched by the *same*
edge, so slowing the clock down does not help. A hold violation at 1 GHz is still a hold
violation at 100 MHz. This is why hold violations are considered more dangerous: you cannot
bin the part at a lower frequency to escape them.

### Slack

**Slack** is the margin by which a path meets or misses its requirement, and it is the
number you will spend most of your life looking at.

```
setup slack = required arrival time − actual arrival time
hold  slack = actual arrival time − required arrival time
```

A **positive** slack means the path passes with room to spare. A **negative** slack is a
violation. Two aggregate metrics matter:

- **WNS** (worst negative slack) — the single worst violating path.
- **TNS** (total negative slack) — the sum of all negative slacks.

They tell you different things. WNS unchanged but TNS improved means optimisation fixed
many small violations without touching the one hard path. WNS improved but TNS worse means
you fixed the headline path and broke several others.

{: .interview }
> *"You fix a setup violation and it creates a hold violation on the same path. What
> happened?"* You made the path faster — upsizing, VT swap, rerouting — and the same path
> that was too slow for setup is now fast enough to race through and violate hold. Setup
> and hold constrain the same path from opposite ends, so any fix that moves delay in one
> direction eats margin in the other. Hold fixes (buffer insertion, downsizing) are usually
> applied afterwards and locally, because delay added for hold costs area and power but
> does not have to sit on the critical setup path.

### Metastability

If a flip-flop's data input changes inside the setup/hold window, the flop can enter a
**metastable** state: its output settles to neither a clean 1 nor a clean 0 for an
unpredictable length of time. Downstream logic then sees an invalid level and can behave
unpredictably.

Within a single clock domain, STA's job is to guarantee this never happens. Across clock
domains, where you cannot guarantee it by construction, you use synchronisers.

---

## Delay, transition, and the things STA computes

### Propagation delay

For a combinational cell, **propagation delay** is the time from an input change to the
output settling at its new stable value. It is not a fixed number per cell — it depends on

- the **input transition time** (a slower input edge produces a later output), and
- the **output load capacitance** (more load takes longer to charge).

This is why Liberty stores delay as a **2-D table** indexed by input slew and output load,
rather than as a single value.

### Transition time and slew
{: #transition-time-and-slew }

**Transition time** is how long a signal takes to move from one logic level to the other.
It is the inverse notion of **slew rate** — a fast slew rate means a short transition time.

![Transition time thresholds]({{ site.baseurl }}/assets/img/transition-time-01.png)

Transition is measured between percentage thresholds of the supply, not between the rails
themselves, because the very beginning and end of a real edge asymptote slowly. A common
definition measures the fall transition as the time to go from 70% to 30% of the logic high
level, with rise and fall thresholds allowed to differ. The exact thresholds come from the
library.

Transition time matters for three reasons:

1. It feeds directly into the **delay** of the next cell.
2. Slow edges increase **short-circuit power** (both networks conduct for longer).
3. Slow edges make a net more vulnerable to
   [crosstalk]({{ site.baseurl }}/docs/signoff/signal-integrity/), because the signal
   spends more time in the undefined region.

Hence `set_max_transition` as a design rule constraint.

### Unateness
{: #unateness }

A timing arc's **unateness** describes whether the direction of an output transition is
predictable from the direction of the input transition.

- **Positive unate** — rising input causes rising output, falling causes falling. An AND
  or OR gate's arcs are positive unate.
- **Negative unate** — rising input causes falling output, and vice versa. Inverters, NAND,
  and NOR arcs are negative unate.
- **Non-unate** — the output direction cannot be determined from the input direction alone;
  it depends on the other inputs. XOR arcs are non-unate.

![Unate arcs]({{ site.baseurl }}/assets/img/unateness-01.png)

This is not trivia. STA needs to know which output edge to follow, and how the *clock*
edge polarity propagates through the clock network. An inverter in the clock path is a
negative unate arc, which is precisely the mechanism by which a design gets a
negative-edge-triggered clock domain.

![Using unateness in the clock path]({{ site.baseurl }}/assets/img/unateness-02.png)

Non-unate arcs in a clock path are a problem, because the tool then has to consider both
polarities — which is why clock networks are built from buffers and inverters, not XORs.

### Fanin and fanout

**Fanout** of a reference register is the set of endpoints its paths reach, through
combinational logic. **Fanin** is the set of startpoints whose paths converge on it.

![Fanin and fanout]({{ site.baseurl }}/assets/img/fanin-and-fanout-01.png)

A large fanout means one driver charging many input capacitances. To do that in reasonable
time it needs high drive strength, which means a physically large cell drawing large
currents — and large currents concentrated in one place is how you get
[electromigration]({{ site.baseurl }}/docs/signoff/physical-verification/#em-electromigration)
and IR drop problems. If the driver is *not* upsized, the transition gets slow and you
collect transition violations instead.

The usual remedies are buffer trees (split the fanout across several drivers) or logic
restructuring. `set_max_fanout` in the SDC constrains this up front.

---

## Sources

- R. Robucci, *Lecture 13 – Timing Analysis*, CMPE Reconfigurable System Design, UMBC — on
  static versus dynamic timing analysis, the absence of test vectors in STA, and the
  setup/hold window around a clock edge.
  <https://eclipse.umbc.edu/robucci/cmpeRSD/Lectures/Lecture13__TimingAnalysis/>
- X. Zhang, *Lecture 13: Timing Analysis, Part 2*, ESE 461, Washington University in
  St. Louis — timing check types and setup/hold slack.
  <https://classes.engineering.wustl.edu/ese461/Lecture/week7b.pdf>
- E. Casas, *Static Timing Analysis*, ELEX 7660 Digital System Design, BCIT/UBC — computing
  setup and hold slack from a delay model.
  <https://people.ece.ubc.ca/edc/7660.jan2018/lec8.pdf>
- *E2ESlack: An End-to-End Graph-Based Framework for Pre-Routing Slack Prediction*, arXiv —
  formal definition of slack as required arrival time minus arrival time, and the sign
  convention for setup versus hold. <https://arxiv.org/pdf/2501.07564>
