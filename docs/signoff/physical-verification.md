---
layout: default
title: Physical Verification
parent: 5. Signoff
nav_order: 1
---

# Physical Verification
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## What gets checked

Once the design is routed, a set of checks decides whether it can actually be manufactured
and whether it still implements the circuit you started with. Broadly:

| Check | Question |
|:--|:--|
| **DRC** | Does the layout obey the foundry's geometric rules? |
| **LVS** | Does the layout match the netlist? |
| **Antenna** | Will any net damage a gate during manufacturing? |
| **EM** | Will any wire degrade over the product's lifetime? |
| **Connectivity** | Are there opens, shorts, or floating nets? |
| **Metal density** | Is the metal fill within the process window? |

---

## DRC (Design Rule Check)

**DRC** verifies that the layout obeys the foundry's geometric rules: minimum spacing,
minimum width, enclosure, density, notch rules, and so on. These rules exist because
lithography and etch have physical limits, and a shape that violates them will not print
reliably.

DRC is run after any step that creates geometry — particularly after
[CTS]({{ site.baseurl }}/docs/implementation/cts/) and after
[routing]({{ site.baseurl }}/docs/implementation/routing/) — and again at signoff.

![DRC check]({{ site.baseurl }}/assets/img/drc-design-rule-check-01.png)

{: .note }
> In-tool DRC checks against the rules in the **technology LEF**, which makes them fast and
> lightweight enough to run repeatedly during implementation. Signoff DRC runs in a
> dedicated physical verification tool against the full foundry rule deck, which is far more
> exhaustive and much slower. Passing in-tool DRC is necessary, not sufficient.

### Checks available in the GUI

| Menu / command | Checks |
|:--|:--|
| `verifyWireGap -wireToWire <distance>` | Wire-to-wire spacing; run after power planning to confirm no stripe was broken |
| Verify → Verify Connectivity (`verifyConnectivity`) | Opens, loops, floating nets, antenna connectivity |
| Verify → Metal Density (`verifyMetalDensity`) | Metal density markers within an area |
| Verify → DRC | All DRC violations, against rules in the tech LEF |
| Verify → Verify Antenna (`verify_antenna`) | Antenna ratio violations |
| Verify → Verify AC Limit (`verifyACLimit`) | Electromigration current limits |
| `verifyWellTap -cell <CELL_NAME> -rule <distance>` | Well tap spacing; run after adding well taps |

![Verification menu]({{ site.baseurl }}/assets/img/drc-design-rule-check-02.png)

### Commands

`set_verify_drc_mode` configures the check — `-check_only <cell>` to check a specific cell,
`-ignore_cell_blockage` to ignore cell blockages, `-disable_rules` to switch off specific
rules. `verify_drc` runs it.

![DRC commands]({{ site.baseurl }}/assets/img/drc-design-rule-check-03.png)

Disabling rules is for isolating a problem during debug, not for making a report look clean.

---

## LVS (Layout Versus Schematic)

**LVS** compares the routed physical layout against the Verilog netlist it was built from,
and reports any difference. It answers a question DRC cannot: the layout might be perfectly
manufacturable and still implement the wrong circuit.

Typical failures:

- **Shorts** — two nets connected that should not be.
- **Opens** — a net that should be connected but is not.
- **Missing or extra instances** — often the aftermath of a botched
  [ECO]({{ site.baseurl }}/docs/signoff/timing-closure/#eco-engineering-change-order).
- **Unconnected power or ground** — commonly a missing `globalNetConnect`.

{: .interview }
> *"What does an LVS check look for? What happens if it fails?"* It extracts a netlist from
> the physical layout and compares it, device by device and net by net, against the source
> netlist. A mismatch means the silicon would not implement the verified design, so it is a
> hard blocker — you cannot tape out on a failing LVS regardless of how good the timing is.
> The usual causes are ECO mistakes, power connection errors, and shorts introduced by
> manual routing edits.

---

## Antenna violations
{: #antenna-violations }

The **antenna effect** — more formally plasma-induced gate oxide damage — is a manufacturing
hazard, not a functional one. It is worth understanding properly because the mechanism
explains why the fixes work.

### The mechanism

During fabrication, metal layers are patterned by **plasma etching**. Plasma contains
energetic ions and radicals, and a partially-built metal segment exposed to it collects
electrical charge. How much charge depends on the segment's surface area.

Now consider a net part-way through the back end of line. The net connects a driver
(a source/drain diffusion) to a load (a transistor **gate**, sitting on very thin gate
oxide). In the finished chip, the diffusion forms a diode that harmlessly conducts or breaks
down non-destructively before the gate oxide can be damaged — so the gate is protected.

But during manufacturing that protection may not exist yet. While M1 is being etched, M2 has
not been formed, so the path back to the protective diffusion may not be complete. The only
discharge path for the accumulated charge is **through the gate oxide** — and if the voltage
gets high enough, the oxide breaks down permanently.

![Antenna effect]({{ site.baseurl }}/assets/img/antenna-violations-01.png)

**Long nets are the risk**, because a larger metal area collects more charge. Foundries
express the rule as an **antenna ratio**: the area of metal connected to a gate divided by
the gate oxide area, which must stay below a foundry limit. The rules come in an antenna
rule file from the foundry.

As gate oxides get thinner with each node, the tolerable ratio drops and the problem gets
harder.

### The fixes

**1. Layer hopping (jumper insertion).** Break a long low-layer route by jumping up to a
higher metal layer part-way along, then coming back down. Because the higher layer is
patterned *later*, the long low-layer segment is split into two shorter pieces at the moment
it is etched, and neither collects enough charge to matter. This is the **default** fix in
most flows, because it costs only routing rather than area.

**2. Antenna diode insertion.** Add a diode cell near the gate, providing a deliberate
discharge path to VSS during manufacturing. Costs area and a small amount of leakage, so it
is generally applied to the violations jumpers could not resolve.

**3. Diode in cell.** Some library cells include a built-in protection diode, which removes
the problem for their inputs at the cost of cell area.

**4. Reducing via area** on the offending connection.

Configuration of the antenna fixing mode is part of
[detail routing]({{ site.baseurl }}/docs/implementation/routing/#configuring-and-reporting).

---

## EM (Electromigration)
{: #em-electromigration }

**Electromigration** is a wear-out mechanism. Conducting electrons transfer momentum to the
metal ions in a wire — the "electron wind" — and over time this literally displaces atoms in
the direction of current flow.

Where the atomic flux diverges, at grain boundaries, vias, or interfaces, the displaced
atoms leave behind **voids** (which grow until the wire opens) or pile up into **hillocks**
(which can short to a neighbouring conductor). Either way, the chip fails — not at test, but
after months or years in the field.

![Electromigration]({{ site.baseurl }}/assets/img/em-electromigration-01.png)

### Black's equation

The standard model for the mean time to failure of an interconnect:

```
MTTF = A · J^(−n) · exp(Ea / kT)
```

where `J` is current density, `Ea` the activation energy, `k` Boltzmann's constant, `T`
absolute temperature, `n` a scaling exponent (commonly between 1 and 2), and `A` a constant
capturing material properties and geometry.

Two consequences you should be able to recite:

1. **Lifetime falls as a power law in current density.** Halving current density buys you
   substantially more than double the lifetime. This is why wider and shorter conductors,
   and parallel current paths, help so much.
2. **Lifetime falls exponentially with temperature.** A hot region of the chip ages far
   faster than a cool one.

There is also a nasty feedback loop: as voids form, the cross-sectional area shrinks, so
current density rises, so IR drop and local heating rise, so degradation accelerates.

### Fixing EM violations

![Fixing EM violations]({{ site.baseurl }}/assets/img/em-electromigration-02.png)

The root cause is usually a driver carrying too much current, which usually means a large
driver feeding a large [fanout]({{ site.baseurl }}/docs/timing/sta-basics/#fanin-and-fanout).
The options:

- **Break up the fanout** — split one big driver into several smaller ones, so no single
  wire carries the whole current.
- **Downsize the driver** where timing permits.
- **Insert buffers** to restructure the net.
- **Widen the wire** or apply an [NDR]({{ site.baseurl }}/docs/implementation/routing/#ndrs-non-default-rules)
  giving it more metal.
- **Use multi-cut vias and via pillars** so current is shared across parallel paths.

### Checking

```tcl
# 1. set a switching activity file
# 2. generate the report
verifyACLimit
# 3. fix
fixACLimitViolation
```

If you have an NDR designed for EM repair, pass it as the `<route_rule>` input to the fix
command.

![verifyACLimit]({{ site.baseurl }}/assets/img/em-electromigration-03.png)

![EM fixing]({{ site.baseurl }}/assets/img/em-electromigration-04.png)

{: .note }
> EM analysis needs realistic **switching activity**. A net that toggles rarely carries
> little average current no matter how big its driver is. Without an activity file, the tool
> assumes defaults, and your EM report describes a design that does not exist.

---

## DPT (Double Patterning Technology)

Below a certain pitch, a single lithographic exposure cannot resolve adjacent features.
**Double patterning** splits the shapes on one layer across **two masks**, so that
neighbouring features are printed in separate exposures and each exposure only has to
resolve features at twice the final pitch.

The two masks are exported separately, and each shape must be assigned to one of them —
conventionally described as **colouring**. Not every layout can be coloured legally: a
pattern that forces two too-close shapes onto the same mask is a **colouring conflict**, and
it is a DRC violation specific to double-patterned layers.

The **Common Coloring Engine** identifies cells and shapes that would violate double
patterning rules, so they can be caught during implementation rather than at signoff.

---

## GDSII and tapeout
{: #gdsii-and-tapeout }

**GDSII** is the stream format in which the finished layout is delivered to the foundry. It
is the final artefact of the whole flow — everything before it exists to produce a GDSII
that is correct.

![Layer map file]({{ site.baseurl }}/assets/img/gdsii-01.png)

A **layer map file** maps layer *names* to layer *numbers*, so that different tools agree on
which physical layer is which. Without a correct map, a GDSII will open with its layers
scrambled.

### Saving a design

| Step | Command / menu | Produces |
|:--|:--|:--|
| Save design database | File → Save Design | `.dat` file and directory |
| Save netlist | `saveNetlist <filename>` | Verilog netlist |
| Save netlist with physical cells | `saveNetlist <filename> -phys` | Netlist including cells added during PD |
| Write LEF abstract | `write_lef_abstract` | LEF abstract of the block |
| Save DEF | `defOut`, or Save → Def | DEF |
| Write SDF | Timing → WriteSDF | Delay annotation for simulation |

The `-phys` variant matters. Physical cells — fillers, taps, end caps, antenna diodes —
were added by the tool and are not in the original synthesis netlist. If the block is going
to be integrated at a higher level, the integrator needs the netlist that includes them.

---

## Sources

- *Antenna Effect in 16nm Technology Node*, eInfochips / Design & Reuse — antenna effect as
  plasma-induced damage, the antenna ratio rule, and fixes by diode, upper-layer routing,
  and via area reduction. <https://www.design-reuse.com/article/61201-antenna-effect-in-16nm-technology-node-/>
- *Dummy Contacts to Mitigate Plasma Charging Damage to Gate Dielectrics*, US Patent
  10,249,621 — why the protective source/drain diode is absent mid-fabrication and present
  in the finished part. <https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/10249621>
- *Antenna Effect*, Wikipedia — overview and the standard list of fixes.
  <https://en.wikipedia.org/wiki/Antenna_effect>
- *Black's Equation for MTTF Due to Electromigration*, Cadence — Black's equation, diffusion
  mechanisms, and the thermal runaway interaction with IR drop.
  <https://resources.system-analysis.cadence.com/blog/msa2020-blacks-equation-for-mttf-due-to-electromigration>
- *Extending Silicon Lifetime: A Review of Design Techniques for Reliable Integrated
  Circuits*, arXiv — the two-stage EM failure mechanism, voids versus hillocks.
  <https://arxiv.org/pdf/2503.21165>
- *Electromigration Concerns Grow In Advanced Packages*, Semiconductor Engineering — why
  shorter, wider interconnects have longer MTTF and the strong temperature dependence.
  <https://semiengineering.com/electromigration-concerns-grow-in-advanced-packages/>
- Silicon Integration Initiative / Cadence, *LEF/DEF 5.8 Language Reference* — FIXEDMASK and
  multi-mask patterning constraints on cell pin shapes.
  <http://coriolis.lip6.fr/doc/lefdef/lefdefref/LEFSyntax.html>
