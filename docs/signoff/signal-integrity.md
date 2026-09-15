---
layout: default
title: Signal Integrity
parent: 5. Signoff
nav_order: 2
---

# Signal Integrity
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## The problem

Nets are not electrically isolated from each other. Two wires running alongside each other
form a capacitor, and that **coupling capacitance** means a voltage change on one net
induces a voltage change on the other.

At older nodes this was a second-order effect. At advanced nodes, wires are thinner, taller,
and packed closer together, so the sidewall coupling between neighbours is a large fraction
of a net's total capacitance. Signal integrity stopped being optional.

The vocabulary is **aggressor** and **victim**: the aggressor is the net that switches, the
victim is the net that suffers the induced disturbance. A net is usually both, to different
neighbours.

---

## Crosstalk

**Crosstalk** is the general term for this coupling-induced interference. Its effect on a
switching victim is a **delay change**, and the direction depends on the relative switching
directions:

| Aggressor switches | Effect on victim |
|:--|:--|
| **Same direction** as the victim | Victim transitions **faster** — delay decreases (speed-up) |
| **Opposite direction** to the victim | Victim transitions **slower** — delay increases (slow-down) |

![Crosstalk]({{ site.baseurl }}/assets/img/crosstalk-01.png)

Both directions are dangerous, and they map onto different checks:

- **Slow-down** eats setup margin. The victim path becomes slower than nominal analysis
  predicted.
- **Speed-up** eats hold margin. The victim path becomes faster, and may now race.

This is why crosstalk analysis has to be run for both setup and hold, and why "it only makes
things faster" is not reassuring.

---

## Glitches

A **glitch** is the other failure mode: instead of perturbing a *switching* victim, an
aggressor perturbs a *quiet* one.

![Glitch]({{ site.baseurl }}/assets/img/glitches-01.png)

The aggressor's transition couples a voltage bump or dip onto the static victim net. If that
excursion is large enough to cross the receiving gate's switching threshold, the victim
momentarily looks like the wrong logic value — and if that momentary wrong value propagates
and gets captured, you have a **functional failure**, not merely a timing one.

Glitches on **clock** and **reset** nets are the most serious case, because a spurious edge
there causes a spurious clock or reset event.

---

## What the tool does automatically

Modern routers do a substantial amount of SI preservation without being asked. The key
mechanism is **timing windows**.

The tool uses the `.lib` timing data to work out *when* each net can switch. Two adjacent
nets only interfere if their switching windows overlap — an aggressor that switches at a
time when the victim is not sensitive cannot hurt it. Where the tool detects nets whose
windows do overlap and whose coupling is significant, it reroutes one of them elsewhere.

This is why SI-aware routing produces different results from timing-blind routing even when
the timing constraints are identical.

---

## Fixing SI problems

![SI fixing methods]({{ site.baseurl }}/assets/img/methods-for-fixing-si-signal-integrity-issues-01.png)

Roughly in order of cost:

**1. Increase spacing.** Apply an
[NDR]({{ site.baseurl }}/docs/implementation/routing/#ndrs-non-default-rules) giving the
victim extra spacing to its neighbours. Coupling capacitance falls with distance, so this is
direct and effective — at the cost of routing resource.

**2. Shield the net.** Route grounded wires on both sides of the victim. The
[shield]({{ site.baseurl }}/docs/implementation/routing/#shielding) absorbs the coupling
that would otherwise reach it. This is the standard treatment for clock nets and other
nets where a glitch would be catastrophic. Expensive in tracks.

**3. Upsize the victim's driver.** A stronger driver holds the victim net more firmly at its
logic level, so the same injected charge produces a smaller voltage excursion. Effective
against glitches specifically.

**4. Downsize or slow the aggressor.** A slower aggressor edge injects current over a longer
period and produces a smaller peak disturbance. Only viable if the aggressor has timing
slack.

**5. Buffer the victim net.** Splitting a long victim into segments reduces the coupled
length per segment.

**6. Reroute.** Move one of the two nets so they no longer run parallel for a long distance.
Long parallel runs are the real culprit; brief crossings couple very little.

**7. Layer change.** Move the net to a layer with more favourable geometry.

{: .interview }
> *"A net is failing due to crosstalk noise. You can't move the net. How do you fix it?"*
> Work on the two endpoints and the aggressor rather than the geometry. Upsize the victim's
> driver so it holds the net harder against injected charge. Downsize or slow the aggressor
> if it has slack. Buffer the victim to shorten the coupled segment. If the victim's receiver
> has a better noise margin available — a higher-threshold cell, say — swap it. Failing all
> that, shield it, which changes the neighbours rather than the net itself.

---

## Analysis commands

| Command | Purpose |
|:--|:--|
| `report_noise` | Text-based noise report |
| `check_noise` | Noise checking, consistency and completeness |
| `setSIMode` | Override default SI analysis settings, e.g. `-enable_glitch_propagation true` |
| `setDelayCalMode` | Consistency and completeness checks; ensures the right set of models is present |

{: .warning }
> SI analysis is only meaningful on **detail-routed** geometry with real extracted coupling
> capacitance. Running it on
> [early global route]({{ site.baseurl }}/docs/implementation/routing/#router-types) results
> is not recommended — EGR does not know where wires actually run, so it cannot know which
> nets are adjacent.

---

## Sources

- *Routing in VLSI Physical Design — Global, Detailed, DRC & SI*, EcrioniX — aggressor and
  victim terminology, glitch propagation thresholds, and the relationship between wire
  density and capacitive coupling. <https://ecrionix.org/physical-design/routing/>
- C. Patel, *Standard Cell Library / Library Exchange Format*, CMPE 641, UMBC — layer
  capacitance per unit area and wire-to-wire coupling modelled in the tech LEF.
  <https://courses.cs.umbc.edu/graduate/CMPE641/Fall08/cpatel2/slides/lect04_LEF.pdf>
- R. Amirtharajah, *CMOS Power Dissipation and Trends*, EEC 216, UC Davis — short-circuit
  current during slow transitions, which is aggravated by crosstalk-induced slew degradation.
  <https://www.ece.ucdavis.edu/~ramirtha/EEC216/W08/lecture1_updated.pdf>
