---
layout: default
title: Glossary
parent: 7. Study and Reference
nav_order: 1
---

# Glossary
{: .no_toc }

Short definitions with links to the page that explains each term properly. Use the search
box for anything not listed here.
{: .fs-5 .fw-300 }

<details open markdown="block">
  <summary>Jump to</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## A

**AOCV (Advanced On-Chip Variation)** — Derating that scales with path depth and distance
rather than applying one flat factor, on the reasoning that variation averages out along
longer paths. → [Variation]({{ site.baseurl }}/docs/timing/variation/#aocv-advanced-ocv)

**Antenna violation** — A manufacturing hazard where a long metal segment accumulates
charge during plasma etching and damages the gate oxide it connects to. →
[Physical verification]({{ site.baseurl }}/docs/signoff/physical-verification/#antenna-violations)

**Aggressor / victim** — In crosstalk analysis, the switching net that induces noise and
the net that suffers it. → [Signal integrity]({{ site.baseurl }}/docs/signoff/signal-integrity/)

## B

**Blockage** — A region where the tool may not place cells (placement blockage) or route
wires (routing blockage). A *partial* placement blockage caps the density allowed inside it
rather than forbidding placement outright. →
[Floorplanning]({{ site.baseurl }}/docs/implementation/floorplanning/)

**Buffer** — A non-inverting cell inserted to drive a large load or long wire, splitting one
big capacitance into two smaller ones. →
[MOSFETs and CMOS]({{ site.baseurl }}/docs/fundamentals/mosfets-and-cmos/#where-delay-and-power-actually-come-from)

## C

**CCOpt** — Cadence's concurrent clock and datapath optimisation engine, which builds the
clock tree and optimises datapath timing together rather than in sequence. →
[CTS]({{ site.baseurl }}/docs/implementation/cts/)

**Cell padding** — Reserved empty space around a placed cell, used to relieve local
congestion or to keep high-current cells away from their neighbours. →
[Placement]({{ site.baseurl }}/docs/implementation/placement/)

**CMOS** — Complementary MOS: logic built from complementary PMOS pull-up and NMOS
pull-down networks. → [MOSFETs and CMOS]({{ site.baseurl }}/docs/fundamentals/mosfets-and-cmos/#cmos-putting-them-together)

**Congestion** — Demand for routing resources in a region exceeding the tracks available
there. Measured via GCell overflow and hotspots. →
[Routing]({{ site.baseurl }}/docs/implementation/routing/#congestion)

**Corner** — A set of physical operating conditions (process, voltage, temperature) plus
the matching extraction settings. →
[Variation]({{ site.baseurl }}/docs/timing/variation/#corners-modes-and-mmmc)

**CPF (Common Power Format)** — A file describing power intent for multi-voltage and
low-power designs. → [Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#cpf-upf-power-intent)

**Crosstalk** — Noise coupled between adjacent nets through their mutual capacitance,
affecting both delay and functional correctness. →
[Signal integrity]({{ site.baseurl }}/docs/signoff/signal-integrity/)

**CTS (Clock Tree Synthesis)** — Building and balancing the physical network that delivers
the clock to every sequential element. → [CTS]({{ site.baseurl }}/docs/implementation/cts/)

## D

**Decap cell** — A physical cell adding local decoupling capacitance to stabilise the supply
during switching. → [Cells]({{ site.baseurl }}/docs/fundamentals/cells/#physical-cells)

**DEF (Design Exchange Format)** — ASCII format describing one specific design's physical
configuration: die area, rows, tracks, placement, pins, nets. →
[Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#def-design-exchange-format)

**Derate** — A multiplier applied to delays to add safety margin for on-chip variation. →
[Variation]({{ site.baseurl }}/docs/timing/variation/#derating)

**Detail route** — The final, DRC-correct routing of every net onto actual tracks and vias,
after global route has planned the approximate paths. →
[Routing]({{ site.baseurl }}/docs/implementation/routing/)

**DFT (Design for Test)** — Logic added specifically to make the manufactured part testable:
scan chains, JTAG, and related structures. →
[Placement]({{ site.baseurl }}/docs/implementation/placement/#scan-chains)

**DPT (Double Patterning Technology)** — Splitting one layer's shapes across two masks so
that features can be packed at a tighter pitch than a single exposure allows. →
[Physical verification]({{ site.baseurl }}/docs/signoff/physical-verification/)

**DRC (Design Rule Check)** — Verification that the layout obeys the foundry's geometric
rules for spacing, width, density, and enclosure. →
[Physical verification]({{ site.baseurl }}/docs/signoff/physical-verification/)

**Drive strength** — How much current a cell can source or sink, set by transistor width.
Higher drive is faster into a given load but larger and presents more input capacitance. →
[Cells]({{ site.baseurl }}/docs/fundamentals/cells/#standard-cells)

**Dynamic power** — Power consumed while a cell is actively switching. →
[Power analysis]({{ site.baseurl }}/docs/signoff/power-analysis/)

## E

**ECO (Engineering Change Order)** — A targeted, incremental change to a design after it has
otherwise been completed. *Premask* ECOs may change any layer; *postmask* ECOs may only
change metal. → [Timing closure]({{ site.baseurl }}/docs/signoff/timing-closure/#eco-engineering-change-order)

**EGR (Early Global Route)** — A fast, approximate global routing pass used to estimate
congestion and parasitics before real routing. →
[Routing]({{ site.baseurl }}/docs/implementation/routing/)

**Electromigration (EM)** — Gradual displacement of metal atoms in wires carrying high
current density, eventually causing opens and failure. →
[Physical verification]({{ site.baseurl }}/docs/signoff/physical-verification/#em-electromigration)

**End cap cell** — A physical cell terminating the end of a row so that well and implant
layers close correctly. → [Cells]({{ site.baseurl }}/docs/fundamentals/cells/#physical-cells)

**Extraction** — Computing the resistance and capacitance of routed interconnect from its
actual geometry, output as SPEF. →
[Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#parasitic-rc-and-extraction)

## F

**False path** — A path that cannot be functionally exercised and is therefore excluded from
timing analysis. → [Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#timing-exceptions)

**Fanout** — The set of endpoints reachable from a given startpoint through combinational
logic; loosely, how many loads a driver feeds. →
[STA basics]({{ site.baseurl }}/docs/timing/sta-basics/#fanin-and-fanout)

**Filler cell** — A physical cell placed in leftover gaps to keep implant and well layers
continuous. → [Cells]({{ site.baseurl }}/docs/fundamentals/cells/#physical-cells)

**Floorplan** — The physical organisation of the block: die and core size, macro positions,
rows, blockages, and pin locations. →
[Floorplanning]({{ site.baseurl }}/docs/implementation/floorplanning/)

## G

**GBA (Graph-Based Analysis)** — Timing analysis that propagates the worst slew and arrival
at each node regardless of path. Fast and pessimistic. →
[Variation]({{ site.baseurl }}/docs/timing/variation/#pba-versus-gba)

**GCell** — A rectangular tile the core is divided into for global routing; routing demand
and capacity are compared per GCell. →
[Routing]({{ site.baseurl }}/docs/implementation/routing/#congestion)

**GDSII** — The stream format in which the finished layout is delivered to the foundry. →
[Physical verification]({{ site.baseurl }}/docs/signoff/physical-verification/#gdsii-and-tapeout)

**Global route** — Planning approximate paths for every net and estimating parasitics,
before detail routing commits to exact geometry. →
[Routing]({{ site.baseurl }}/docs/implementation/routing/)

**Guide / region / fence** — Placement constraints of increasing strictness, confining a
module's cells to an area. →
[Floorplanning]({{ site.baseurl }}/docs/implementation/floorplanning/#guides-regions-and-fences)

## H

**Halo** — A keep-out margin around a macro preventing standard cells (and sometimes
routing) from crowding its edges. →
[Floorplanning]({{ site.baseurl }}/docs/implementation/floorplanning/)

**Hold check** — Verification that data remains stable long enough after the clock edge; a
race between two paths launched by the same edge. →
[STA basics]({{ site.baseurl }}/docs/timing/sta-basics/#setup-and-hold)

**H-tree** — A clock distribution structure with geometrically matched branch lengths,
giving low skew at the cost of requiring regular, unblocked area. →
[CTS]({{ site.baseurl }}/docs/implementation/cts/)

**HVT (High-VT)** — A high-threshold cell variant: slow, low leakage. →
[MOSFETs and CMOS]({{ site.baseurl }}/docs/fundamentals/mosfets-and-cmos/#threshold-voltage-flavours-vt)

## I

**Insertion delay** — The time for the clock to propagate from its definition point through
the clock tree to a sink. →
[Clocks]({{ site.baseurl }}/docs/timing/clocks/#clock-latency-insertion-delay)

**IR drop** — Voltage lost across the resistance of the power distribution network, reducing
the supply seen by cells and slowing them down. →
[Power analysis]({{ site.baseurl }}/docs/signoff/power-analysis/)

**ITF (Interconnect Technology Format)** — Foundry-supplied description of the layer stack
used for accurate parasitic extraction. →
[Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#parasitic-rc-and-extraction)

## J

**JTAG** — A standard boundary-scan debug and test interface; its cells are normally placed
near the core boundary. → [Placement]({{ site.baseurl }}/docs/implementation/placement/)

## L

**Layer hopping** — Fixing an antenna violation by breaking a long low-layer net and routing
part of it on a higher layer. →
[Physical verification]({{ site.baseurl }}/docs/signoff/physical-verification/#antenna-violations)

**Leakage power** — Current flowing from VDD to VSS even when a cell is not switching. →
[Power analysis]({{ site.baseurl }}/docs/signoff/power-analysis/)

**LEF (Library Exchange Format)** — ASCII format describing the physical abstract view of a
library: layers, vias, sites, and cell macros. →
[Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#lef-library-exchange-format)

**Liberty (`.lib`)** — Timing and power characterisation of every cell, at one PVT corner. →
[Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#lib-liberty-timing-libraries)

**Lockup latch** — A latch inserted to add half a cycle of delay, chiefly on scan paths and
clock domain crossings. → [Clocks]({{ site.baseurl }}/docs/timing/clocks/#lockup-latches)

**LVS (Layout Versus Schematic)** — Verification that the routed layout matches the netlist
it was built from. →
[Physical verification]({{ site.baseurl }}/docs/signoff/physical-verification/)

**LVT (Low-VT)** — A low-threshold cell variant: fast, high leakage. →
[MOSFETs and CMOS]({{ site.baseurl }}/docs/fundamentals/mosfets-and-cmos/#threshold-voltage-flavours-vt)

## M

**Macro** — A large pre-built block (SRAM, PLL, IP) placed as a single unit and treated as
an obstruction by placement and routing. →
[Cells]({{ site.baseurl }}/docs/fundamentals/cells/#macros)

**Metastability** — A flip-flop output that settles to neither logic level for an
unpredictable time, caused by data changing inside the setup/hold window. →
[STA basics]({{ site.baseurl }}/docs/timing/sta-basics/#metastability)

**MMMC (Multi-Mode Multi-Corner)** — Analysing the design across several combinations of
functional mode and operating corner simultaneously. →
[Variation]({{ site.baseurl }}/docs/timing/variation/#corners-modes-and-mmmc)

**Mode** — The functional state the design is in: functional, scan shift, test, low power. →
[Variation]({{ site.baseurl }}/docs/timing/variation/#corners-modes-and-mmmc)

**Multicycle path** — A path legitimately allowed more than one clock cycle to propagate. →
[Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#timing-exceptions)

**Multi-cut via** — Several via cuts in parallel between the same two layers, lowering
resistance and improving electromigration robustness. →
[Routing]({{ site.baseurl }}/docs/implementation/routing/)

## N

**NDR (Non-Default Rule)** — A routing rule applied to selected nets giving them extra
width, extra spacing, or shielding. →
[Routing]({{ site.baseurl }}/docs/implementation/routing/)

**Netlist** — A textual description of instances and their connectivity, output by
synthesis. → [Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#gate-level-netlist)

**NMOS** — A transistor that conducts when its gate is high. Passes a strong 0. →
[MOSFETs and CMOS]({{ site.baseurl }}/docs/fundamentals/mosfets-and-cmos/#nmos-and-pmos)

## O

**OCV (On-Chip Variation)** — Variation in cell behaviour between locations on the same die.
→ [Variation]({{ site.baseurl }}/docs/timing/variation/#ocv-on-chip-variation)

**Obstruction (OBS)** — A region inside a cell or macro footprint where the router must not
route, declared in LEF. →
[Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#lef-library-exchange-format)

**Overflow** — More nets needing to cross a GCell than it has tracks for. →
[Routing]({{ site.baseurl }}/docs/implementation/routing/#congestion)

## P

**Patch wire** — A small extra piece of metal added by the detail router to fix a notch or
minimum-area violation. → [Routing]({{ site.baseurl }}/docs/implementation/routing/)

**Path group** — A named bucket of timing paths, optimised and reported separately so one
catastrophic group does not starve the others. →
[Timing closure]({{ site.baseurl }}/docs/signoff/timing-closure/#path-groups)

**PBA (Path-Based Analysis)** — Timing analysis that recomputes slew along each specific
path, removing GBA's pessimism at the cost of runtime. →
[Variation]({{ site.baseurl }}/docs/timing/variation/#pba-versus-gba)

**PDN (Power Distribution Network)** — The rings, stripes, rails, and vias delivering VDD and
VSS throughout the block. →
[Power planning]({{ site.baseurl }}/docs/implementation/power-planning/)

**Pin density** — How many pins must be accessed per unit area. At advanced nodes this
limits placement more often than cell density does. →
[Cells]({{ site.baseurl }}/docs/fundamentals/cells/#routing-tracks)

**PMOS** — A transistor that conducts when its gate is low. Passes a strong 1. →
[MOSFETs and CMOS]({{ site.baseurl }}/docs/fundamentals/mosfets-and-cmos/#nmos-and-pmos)

**PPA** — Power, performance, and area: the three quantities every implementation decision
trades against. →
[Timing closure]({{ site.baseurl }}/docs/signoff/timing-closure/)

**Propagated clock** — A clock whose real network delays are included in analysis, as
opposed to an ideal clock. →
[Clocks]({{ site.baseurl }}/docs/timing/clocks/#the-ideal-clock-tree)

**Propagation delay** — Time from an input change to the output settling, dependent on input
slew and output load. →
[STA basics]({{ site.baseurl }}/docs/timing/sta-basics/#propagation-delay)

**PVT** — Process, voltage, temperature: the axes defining an operating corner. →
[Variation]({{ site.baseurl }}/docs/timing/variation/#corners-modes-and-mmmc)

## R

**Rail** — A thin power or ground wire on a low metal layer running along a row, feeding
individual standard cells. →
[Power planning]({{ site.baseurl }}/docs/implementation/power-planning/)

**Rail analysis** — Analysis of the power distribution network's robustness, principally IR
drop. → [Power analysis]({{ site.baseurl }}/docs/signoff/power-analysis/)

**Row** — A horizontal strip of the core, one site tall, into which standard cells are
placed. → [Cells]({{ site.baseurl }}/docs/fundamentals/cells/#sites-and-rows)

## S

**Scan chain** — Flip-flops stitched into a shift register so internal state can be shifted
in and out for test. →
[Placement]({{ site.baseurl }}/docs/implementation/placement/#scan-chains)

**Scan reordering** — Re-sequencing a scan chain during physical design to shorten its
routing, allowed because chain order is not functionally significant. →
[Placement]({{ site.baseurl }}/docs/implementation/placement/#scan-chains)

**SDC (Synopsys Design Constraints)** — The Tcl-based timing constraint format. →
[Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#sdc-synopsys-design-constraints)

**SDF (Standard Delay Format)** — Interchange format for cell and interconnect delays,
typically back-annotated into simulation. →
[Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#sdf-standard-delay-format)

**Setup check** — Verification that data arrives early enough before the capture clock edge.
→ [STA basics]({{ site.baseurl }}/docs/timing/sta-basics/#setup-and-hold)

**Shielding** — Routing grounded wires alongside a sensitive net to isolate it from
crosstalk. → [Routing]({{ site.baseurl }}/docs/implementation/routing/)

**Site** — The smallest placement unit in the floorplan grid; every cell footprint is an
integer number of sites. →
[Cells]({{ site.baseurl }}/docs/fundamentals/cells/#sites-and-rows)

**Skew** — The difference in arrival time between signals; *clock skew* is the spread of
arrival times at clock tree endpoints. →
[Clocks]({{ site.baseurl }}/docs/timing/clocks/#skew)

**Skew group** — A set of clock endpoints the tool is asked to balance against each other,
as distinct from the physical clock tree itself. →
[CTS]({{ site.baseurl }}/docs/implementation/cts/)

**Slack** — Margin by which a path meets its requirement; negative slack is a violation. →
[STA basics]({{ site.baseurl }}/docs/timing/sta-basics/#slack)

**Slew rate** — How quickly a signal transitions; the inverse notion of transition time. →
[STA basics]({{ site.baseurl }}/docs/timing/sta-basics/#transition-time-and-slew)

**SPEF (Standard Parasitic Exchange Format)** — The file carrying extracted R and C values
to the timing tool. →
[Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#the-extraction-tool)

**Spare cell** — Unused logic distributed through the design, available to be rewired during
a postmask ECO. → [Cells]({{ site.baseurl }}/docs/fundamentals/cells/#spare-cells)

**SRoute (Special Route)** — Routing of power and ground connections, as distinct from
signal routing. → [Power planning]({{ site.baseurl }}/docs/implementation/power-planning/)

**STA (Static Timing Analysis)** — Vectorless, exhaustive timing verification based on the
structure of the design. → [STA basics]({{ site.baseurl }}/docs/timing/sta-basics/)

**Standard cell** — A pre-laid-out, pre-characterised logic block of fixed height, the basic
unit of placement. → [Cells]({{ site.baseurl }}/docs/fundamentals/cells/#standard-cells)

**Stripe** — A wide power or ground wire on an upper metal layer, part of the power mesh. →
[Power planning]({{ site.baseurl }}/docs/implementation/power-planning/)

**SVT / RVT (Standard/Regular-VT)** — The middle threshold voltage variant. →
[MOSFETs and CMOS]({{ site.baseurl }}/docs/fundamentals/mosfets-and-cmos/#threshold-voltage-flavours-vt)

## T

**Tapeout** — Delivering the final verified layout to the foundry for manufacture. →
[Physical verification]({{ site.baseurl }}/docs/signoff/physical-verification/#gdsii-and-tapeout)

**Technology LEF** — The LEF file holding process and routing rules, as opposed to cell
abstracts. → [Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#lef-library-exchange-format)

**`.tlf`** — A legacy Cadence timing library format, superseded by Liberty. →
[Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#lib-liberty-timing-libraries)

**TNS (Total Negative Slack)** — The sum of all negative slacks in the design. →
[STA basics]({{ site.baseurl }}/docs/timing/sta-basics/#slack)

**Transition time** — How long a signal takes to move between logic levels, measured between
percentage thresholds. →
[STA basics]({{ site.baseurl }}/docs/timing/sta-basics/#transition-time-and-slew)

## U

**Unateness** — Whether an output transition direction is predictable from the input
transition direction. →
[STA basics]({{ site.baseurl }}/docs/timing/sta-basics/#unateness)

**Uncertainty** — Margin declared in SDC that absorbs jitter, estimated skew, and extra
pessimism. → [Clocks]({{ site.baseurl }}/docs/timing/clocks/#clock-uncertainty)

**Useful skew** — Deliberately unbalancing the clock tree to move timing margin from a path
with slack to one without. →
[Clocks]({{ site.baseurl }}/docs/timing/clocks/#useful-skew-and-time-borrowing)

**Utilisation** — The fraction of available area occupied by cells and macros. →
[Cells]({{ site.baseurl }}/docs/fundamentals/cells/#utilisation)

## V

**Via ladder / via stack / via pillar** — Structures connecting a net across several metal
layers; pillars use multiple parallel cuts to spread current and resist electromigration. →
[Routing]({{ site.baseurl }}/docs/implementation/routing/)

**View** — One pairing of a constraint mode with a delay corner. →
[Variation]({{ site.baseurl }}/docs/timing/variation/#corners-modes-and-mmmc)

## W

**Well tap** — A physical cell tying the well and substrate to the supply rails at regular
intervals, preventing latch-up. →
[Cells]({{ site.baseurl }}/docs/fundamentals/cells/#physical-cells)

**Wire load model** — A pre-placement statistical estimate of wire length from fanout, used
before any physical information exists. →
[Design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#parasitic-rc-and-extraction)

**WNS (Worst Negative Slack)** — The single worst violating path in the design. →
[STA basics]({{ site.baseurl }}/docs/timing/sta-basics/#slack)
