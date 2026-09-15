---
layout: default
title: Learning Plan and Further Reading
parent: 7. Study and Reference
nav_order: 3
---

# Learning Plan and Further Reading
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## A three-track approach

Physical design is hard to learn from one direction alone. Theory without tool time stays
abstract; tool time without theory becomes command memorisation. The plan that works is to
run three tracks in parallel.

### 1. Theory

Build the mental model of what the tool is trying to do.

- **Static Timing Analysis for Nanometer Designs: A Practical Approach** (Bhasker &
  Chadha) — the standard reference for STA. Covers timing arcs, constraints, corners,
  variation, and crosstalk-aware analysis in detail.
- **CMOS VLSI Design: A Circuits and Systems Perspective** (Weste & Harris) — the device and
  circuit background underneath [Part 1]({{ site.baseurl }}/docs/fundamentals/).
- University course notes are underrated and freely available. The lecture material cited
  throughout this wiki — UMBC CMPE 641, UC Davis EEC 216, WashU ESE 461, UMN EE 5323 — is
  worth reading directly rather than only in summary.

### 2. Practice

Get a design through the flow. Nothing else teaches what congestion actually looks like.

- Run a small block end to end and make every mistake once.
- **[ASIC Design Roadmap](https://github.com/abdelazeem201/ASIC-Design-Roadmap)** — a
  structured path through the material with resources at each stage.
- **[Basic Static Timing Analysis](https://github.com/abdelazeem201/Basic-Static-Timing-Analysis)** —
  hands-on STA exercises.
- Open-source flows (OpenROAD, OpenLane) with an open PDK such as SKY130 let you run a
  complete RTL-to-GDSII flow without a commercial licence. The tools differ from Innovus in
  vocabulary, not in concept.

### 3. Big picture

Carry one design all the way through, so the stages connect to each other rather than
existing as separate exercises.

- **A RISC-V core** is the standard choice, and a good one: small enough to finish, complex
  enough to be interesting, and with plenty of reference implementations to compare against.
- The goal is to take a single design through the **entire chip design flow** — RTL,
  verification, synthesis, DFT insertion, place and route, signoff, GDSII — rather than
  treating physical design as an isolated stage.

---

## Certifications

The **Cadence Innovus Block Implementation** certification track is worth considering if you
have tool access. Its labs are structured around exactly the flow in
[Part 4]({{ site.baseurl }}/docs/implementation/) and
[Part 5]({{ site.baseurl }}/docs/signoff/), and working through them gives you the repeated
tool exposure that is otherwise hard to get outside a job.

---

## Reading this wiki in order

If you are starting from scratch:

1. [Fundamentals]({{ site.baseurl }}/docs/fundamentals/) — skim if you have taken a VLSI
   course, but read the section on where delay and power come from.
2. [The flow and its inputs]({{ site.baseurl }}/docs/flow/) — get the map before the detail.
3. [Timing]({{ site.baseurl }}/docs/timing/) — the hardest part, and the part everything
   else serves.
4. [Implementation]({{ site.baseurl }}/docs/implementation/) — read alongside actually
   running a block if you can.
5. [Signoff]({{ site.baseurl }}/docs/signoff/) — read when you have something to sign off.
6. [Interview questions]({{ site.baseurl }}/docs/reference/interview-questions/) — use as a
   self-test once you have been through the rest.
