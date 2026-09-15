---
layout: default
title: Power and Rail Analysis
parent: 5. Signoff
nav_order: 3
---

# Power and Rail Analysis
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## Two different questions

These get confused constantly, so start here:

**Power analysis** asks *how much power does this design consume?* It is about the design's
demand — switching, internal, and leakage power, per instance and in total.

**Rail analysis** asks *can the power distribution network actually deliver it?* It is about
the supply — IR drop across the
[PDN]({{ site.baseurl }}/docs/implementation/power-planning/), and whether every cell
actually sees enough voltage.

You need both, and rail analysis needs the output of power analysis as an input.

---

## Where power goes

Total power splits into components, and they behave very differently.

### Switching power

Power spent **charging and discharging load capacitance** as nets toggle. Every time a net
goes from 0 to 1, charge is pulled from the supply into the load capacitance; going back to
0 dumps it to ground.

Because CMOS inputs are essentially capacitive, this is the dominant dynamic component. It
scales with supply voltage squared, with capacitance, and with switching frequency — which
is why reducing voltage is the most powerful dynamic power lever available, and why
[clock gating]({{ site.baseurl }}/docs/implementation/cts/) (which reduces effective
frequency on idle logic) saves so much.

### Internal switching power

Power dissipated **inside** a cell when its inputs change: charging internal nodes, and the
brief **short-circuit** current that flows while both the pull-up and pull-down networks are
momentarily conducting during a transition.

It depends on the cell itself, and it is characterised in the `.lib`. It is also why slow
input transitions are expensive — a sluggish edge keeps both networks partly on for longer.

### Leakage power

Current that flows from VDD to VSS even when nothing is switching. Transistors are
imperfect switches: subthreshold conduction, gate tunnelling through the thin oxide, and
reverse-biased junction leakage all contribute.

Leakage used to be negligible. As geometries shrank and oxides thinned, it grew to be a
serious fraction of total power in many designs. It also **rises with temperature**, which
creates a feedback loop: hotter chip, more leakage, more heat.

Leakage is what
[VT swapping]({{ site.baseurl }}/docs/fundamentals/mosfets-and-cmos/#threshold-voltage-flavours-vt)
trades against speed.

| Component | Scales with | Reduced by |
|:--|:--|:--|
| Switching | V², capacitance, activity | Lower voltage, clock gating, smaller cells, shorter wires |
| Internal | Transition rate, input slew | Faster edges, smaller cells |
| Short-circuit | Input slew | Faster edges |
| Leakage | Cell count, temperature, VT | HVT cells, power gating, fewer/smaller cells |

---

## Power analysis

![Power analysis results]({{ site.baseurl }}/assets/img/report-power-analysis-01.png)

```tcl
set_power_analysis_mode \
  -method static \
  -analysis_view dtmf_view_setup \
  -corner max \
  -create_binary_db true \
  -write_static_currents true \
  -honor_negative_energy true \
  -ignore_control_signals true
```

The two decisions in that command:

**Static or dynamic?** *Static* (sometimes called average or vectorless) power analysis uses
statistical switching activity to compute average power. *Dynamic* analysis uses actual
simulation activity over time and captures peak power and transients. Static is fast and
adequate for average power budgets; dynamic is what you need for peak current and for
dynamic IR drop.

**Which view?** Power is corner-dependent — leakage in particular varies enormously between
corners — so you must state which
[analysis view]({{ site.baseurl }}/docs/timing/variation/#corners-modes-and-mmmc) you are
measuring.

### Switching activity

Power numbers are only as good as the activity data behind them.

| Command | Purpose |
|:--|:--|
| `read_activity_file` | Read measured switching activity, e.g. a SAIF file from simulation |
| `set_default_switching_activity` | Set assumed activity for nets not explicitly annotated |
| `report_power` | Report the results |

A SAIF file from a realistic workload is worth a great deal more than default assumptions.
Without it you are reporting power for an imaginary traffic pattern — which also undermines
[EM analysis]({{ site.baseurl }}/docs/signoff/physical-verification/#em-electromigration),
since that depends on the same activity data.

`report_power` can include clock pin power, which you generally want, since the clock
network is often the largest single power consumer in the block.

---

## Rail analysis
{: #rail-analysis }

**Rail analysis** evaluates the power distribution network: how much voltage is lost between
the supply sources and each cell.

The output is an **IR drop map** over the die, with warmer colours indicating larger drop.

![IR drop map]({{ site.baseurl }}/assets/img/report-rail-analysis-01.png)

### Reading the map

The example above shows the characteristic pattern. The worst regions — highlighted red —
are those **furthest from any VDD source** and **sitting between two macros**. Both factors
compound: distance means more series resistance in the path from the supply, and the macros
block stripes from crossing, so there is no shorter alternative path. The result is a region
where cells see meaningfully reduced supply and therefore run slower than the timing
analysis assumed.

That is a floorplan and power planning finding, not a routing one. Fixing it means adding
stripes, widening them, improving the via stacks in that area, or moving the macros.

### Running it

Rail analysis needs more collateral than most steps:

1. The **corner and mode** to analyse.
2. The **nominal voltage and violation threshold**.
3. **Power analysis results** — current data per instance.
4. A **power pad file** (`.pp`) describing where supply enters the die.
5. An **extraction tech file** (`.tch`) for parasitic information about the PDN metal.

```tcl
set_rail_analysis_mode -method era_static -power_switch_eco false ...
create_power_pads -net VDD -vsrc_file dtmf.pp
set_pg_nets -net VDD -voltage 0.9 -threshold 0.81
set_power_data -format current -scale 1 run1/static_VDD.ptiavg
set_power_pads -net VDD -format xy -file dtmf.pp
analyze_rail -type net -results_directory ./run1 VDD
read_power_rail_results -power_db run1/power.db -rail_directory run1/VDD_25C_avg_1
set_power_rail_display -plot ir
```

Note `set_pg_nets -net VDD -voltage 0.9 -threshold 0.81` — nominal 0.9 V with a violation
threshold at 0.81 V, i.e. flagging anything dropping more than 10%. The acceptable budget
is a project decision; 5–10% is a common range, and it has to be consistent with the voltage
your timing libraries were characterised at.

{: .warning }
> **If your `.lib` does not match your operating voltage, your timing is wrong.** Liberty
> delay tables are characterised at a specific voltage. If rail analysis shows cells
> operating 8% below nominal but your timing runs against a nominal-voltage library, you are
> signing off on delays that the silicon will not deliver. Either carry a library
> characterised at the drooped voltage, or budget the gap explicitly in
> [uncertainty]({{ site.baseurl }}/docs/timing/clocks/#clock-uncertainty).

### Static versus dynamic IR drop

**Static IR drop** uses average currents and finds the structurally weak parts of the grid.
It is what you run first, and it catches gross problems.

**Dynamic IR drop** uses time-varying currents from simulation and captures transient
droop — the instantaneous sag when a large number of cells switch together. It is the harder
and more realistic analysis, and it is where
[decap cells]({{ site.baseurl }}/docs/fundamentals/cells/#physical-cells) and the argument
against a perfectly balanced clock tree both come from.

---

## Sources

- R. Amirtharajah, *CMOS Power Dissipation and Trends*, EEC 216, UC Davis — the four
  components of CMOS power: dynamic, short-circuit, static, and leakage (subthreshold,
  junction, and gate tunnelling).
  <https://www.ece.ucdavis.edu/~ramirtha/EEC216/W08/lecture1_updated.pdf>
- C. H. Kim, *CMOS Inverter: Power Dissipation and Sizing*, EE 5323, University of
  Minnesota — derivation of switching energy as a function of load capacitance and supply
  voltage. <http://people.ece.umn.edu/~kia/Courses/EE5323/Slides/Lect_04_Inverter2.pdf>
- *Power Dissipation of a CMOS Inverter*, All About Circuits — leakage growing towards
  dynamic power as features shrink, and static power increasing with temperature.
  <https://www.allaboutcircuits.com/technical-articles/power-dissipation-of-a-cmos-inverter/>
- *CMOS Inverter — Power and Energy Consumption*, Technobyte — gate tunnelling through
  thinner oxides as a static power component.
  <https://technobyte.org/cmos-inverter-power-energy-consumption/>
- *Black's Equation for MTTF Due to Electromigration*, Cadence — the IR drop and temperature
  feedback loop. <https://resources.system-analysis.cadence.com/blog/msa2020-blacks-equation-for-mttf-due-to-electromigration>
