---
layout: default
title: Innovus Command Reference
parent: 6. Tools
nav_order: 2
---

# Innovus Command Reference
{: .no_toc }

An index of commands by flow stage, plus the database query mechanism. Syntax changes
between releases — use `<command> -help` in the shell for authoritative options.
{: .fs-5 .fw-300 }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## Getting help

![Getting help in the shell]({{ site.baseurl }}/assets/img/innovus-commands-01.png)

Every command supports `-help` for a brief description of its parameters. For settings-style
commands, there is usually a matching getter: `setPlaceMode` / `getPlaceMode`,
`setDelayCalMode` / `getDelayCalMode`, and so on. Checking the current value before changing
it saves a lot of confusion.

---

## By flow stage

### Setup and import

| Command | Purpose |
|:--|:--|
| `source design.globals` | Load design collateral settings |
| `init_design` | Initialise the design |
| `checkDesign -netlist` | Verify inputs were read correctly |
| `setDelayCalMode` | Global delay calculation parameters — accuracy level, corner-based analysis, high fanout and input slew sensitivity handling |
| `getDelayCalMode` | Report current delay calculation settings |
| `create_library_set` | Bundle `.lib` files into a named set |
| `create_rc_corner` | Define extraction conditions |
| `create_delay_corner` | Combine a library set and an RC corner |
| `create_constraint_mode` | Name a set of SDC files |
| `create_analysis_view` | Pair a constraint mode with a delay corner |
| `set_analysis_view` | Declare active views for setup and hold |

### Floorplanning

| Command | Purpose |
|:--|:--|
| `create_relative_floorplan` | Lock components into relative positions |
| `init_io_file` | Read an I/O assignment file |
| `defIn` | Read a DEF file |
| `assignIoPins -pin *` | Auto-assign pin locations on the core margin |
| `verifyWireGap -wireToWire <dist>` | Check wire-to-wire spacing |

### Power planning

| Command | Purpose |
|:--|:--|
| `addRing` | Power rings — `-width`, `-spacing`, `-offset` |
| `addStripe` | Power stripes across the core |
| `editPowerVia` | Via stacks between grid layers |
| `globalNetConnect` | Connect global power/ground nets to instance pins |
| `sroute` | Special (power) routing |

### Placement

| Command | Purpose |
|:--|:--|
| `setPlaceMode` | Configure placement — call before optimisation |
| `getPlaceMode` | Report current placement settings |
| `place_connected` | Place logic near a specified attractor |
| `place_opt_design` | Placement interleaved with setup optimisation |
| `reportPlacementDensity` | Placement density report |

### Design for test

| Command | Purpose |
|:--|:--|
| `specifyScanCell` | Identify scan cells |
| `specifyScanChain` | Define a scan chain |
| `setScanReorderMode` | Configure scan reordering |
| `scanReorder` | Reorder chains for routability |
| `deleteScanChain` | Remove a chain definition |

### Clock tree synthesis

| Command | Purpose |
|:--|:--|
| `check_design -type cts` | Verify the clock tree definition before optimising |
| `set_ccopt_property` | Configure CTS — `cell_halo_x`/`y`, `auto_limit_insertion_delay_factor`, and many others |
| `clock_opt_design` | Build, route, and optimise the clock tree |
| `clock_opt_design -cts` | Build a balanced / zero-skew tree |
| `report_clock_timing -type skew` | Skew report |
| `report_ccopt_clock_tree_structure` | Report the tree structure built |
| `show_ccopt_cell_name_info` | Clock tree cell naming conventions |

### Routing

| Command | Purpose |
|:--|:--|
| `setDesignMode` | Top and bottom routing layers |
| `setRouteMode` | Routing effort, direction, group attributes |
| `setAttribute` | Per-net attributes, including NDRs |
| `earlyGlobalRoute` | Fast approximate global route |
| `routeDesign` | Full routing |
| `routeDesign -trackOpt` | Track optimisation and post-global-route extraction |
| `routeDesign -globalDetail` | Global plus detail routing |
| `route_opt_design` | Routing with setup and hold optimisation |
| `add_tracks` | Override automatically created tracks |
| `report_route` | Routing statistics including via types |
| `reportWirePath -start <term> -end <term>` | Trace a connection's path |
| `reportCongestion` | Congestion metrics |
| `reportRouteTypeConstraints` | Via pillar count |

### Wire editing

| Command | Purpose |
|:--|:--|
| `setEditMode` | Wire editing properties |
| `editAddRoute` | Add a wire at coordinates |
| `editSelect` / `editDeselect` | Select wires by region, property, or name |
| `editDelete` | Delete nets and vias; `-regular_wire_with_drc` deletes violating wires |
| `editCutWire` | Trim a wire |
| `editResize` | Change wire width or properties |

### Timing analysis

| Command | Purpose |
|:--|:--|
| `timeDesign` | Report timing across active views |
| `timeDesign -prePlace` | Zero-wire-load constraint sanity check |
| `extractRC` | Run parasitic extraction |
| `setOptMode` | Optimisation settings — `-opt_all_end_points`, `-opt_area_recovery_setup_target_slack` |
| `optDesign -postCTS` | Post-CTS setup optimisation |
| `optDesign -postCTS -hold` | Post-CTS hold optimisation |
| `optDesign -postRoute -setup -hold` | Post-route optimisation |
| `group_path` | Create a path group |

### Signal integrity and power

| Command | Purpose |
|:--|:--|
| `report_noise` | Text noise report |
| `check_noise` | Noise checking |
| `setSIMode` | SI analysis settings, e.g. `-enable_glitch_propagation true` |
| `set_power_analysis_mode` | Configure power analysis |
| `read_activity_file` | Read switching activity, e.g. SAIF |
| `set_default_switching_activity` | Activity for unannotated nets |
| `report_power` | Power report |
| `set_rail_analysis_mode` | Configure rail analysis |
| `analyze_rail` | Run rail analysis |
| `create_power_pads` | Define supply entry points |
| `set_pg_nets` | Nominal voltage and violation threshold |
| `all_fanin` / `all_fanout` | Report fanin/fanout of a structure |

### Verification

| Command | Purpose |
|:--|:--|
| `set_verify_drc_mode` | DRC check configuration — `-check_only`, `-ignore_cell_blockage`, `-disable_rules` |
| `verify_drc` | Run DRC |
| `verifyConnectivity` | Opens, loops, floating nets |
| `verifyMetalDensity` | Metal density markers |
| `verify_antenna` | Antenna ratio violations |
| `verifyACLimit` | Electromigration current limits |
| `fixACLimitViolation` | Fix EM violations |
| `verifyWellTap -cell <name> -rule <dist>` | Well tap spacing |

### ECO

| Command | Purpose |
|:--|:--|
| `setEcoMode` | Permitted ECO change types |
| `ecoAddRepeater` | Insert a buffer or inverter |
| `ecoChangeCell` | Swap a cell |
| `attachTerm` / `detachTerm` | Connect/disconnect a cell terminal |
| `ecoPlace` | Place ECO cells (maps to spares in postmask mode) |
| `ecoDesign` | Run ECO; can be restricted to specific layers |
| `loadECO` | Read a file of ECO directives |

### Export

| Command | Purpose |
|:--|:--|
| `saveDesign <name>` | Save the design database |
| `saveNetlist <file>` | Write out the netlist |
| `saveNetlist <file> -phys` | Netlist including physical cells |
| `write_lef_abstract` | Write a LEF abstract of the block |
| `defOut` | Write DEF |

---

## Querying the database with `dbGet`

`dbGet` navigates the design database directly. It is the most useful debugging tool in the
shell, because it lets you ask questions no report was written to answer.

![dbGet]({{ site.baseurl }}/assets/img/innovus-commands-02.png)

The database is reached through two root pointers:

- **`top`** — everything about the design.
- **`head`** — manufacturing and technology information (the tech file).

### Discovering what is available

```tcl
dbGet selected           ;# the selected object(s)
dbGet selected.?         ;# list the attributes of the selected object
dbGet selected.??        ;# list attributes with their values
dbSchema inst            ;# the schema for instance objects
```

![dbGet selected]({{ site.baseurl }}/assets/img/innovus-commands-03.png)

![dbSet]({{ site.baseurl }}/assets/img/innovus-commands-04.png)

![dbSchema]({{ site.baseurl }}/assets/img/innovus-commands-05.png)

![Schema browsing]({{ site.baseurl }}/assets/img/innovus-commands-06.png)

`dbSet` changes an attribute on an object or set of objects.

### Useful floorplan queries

Select all RAMs and hard macros:

```tcl
select_obj [dbGet -p2 top.insts.cell.baseClass block]
```

The GUI may need refreshing to show the selection.

Report the area of a named macro:

```tcl
dbGet [dbGet -p2 top.insts.cell.name <macro_cell_name>].area
```

Check macro orientations:

```tcl
set macro [dbGet -p2 top.insts.cell.baseClass block]
foreach inst $macro {
    puts "[dbGet $inst.name] [dbGet $inst.orient]"
}
```

Design and core boundary of a rectangular floorplan:

```tcl
dbGet top.fplan.boxes
dbGet top.fplan.coreBox
```

For a rectilinear boundary, combine the boxes into a polygon:

```tcl
set designShape [dbShape -output polygon [dbGet top.fplan.boxes]]
```

Report I/O port direction, layer, and location:

```tcl
foreach term [dbGet top.terms] {
    puts "Pin: [dbGet $term.name] ([dbGet $term.inOutDir])"
    foreach pin [dbGet $term.pins] {
        foreach layerShape [dbGet $pin.layerShapeShapes] {
            puts "  Layer: [dbGet $layerShape.layer.extName] - Rects: [dbGet $layerShape.shapes.rect]"
        }
    }
}
```

List the metal layers of a routing blockage:

```tcl
dbGet [dbGet -p top.fplan.rBlkgs.name <routing_blockage_name>].layer.name
```

Convert placement blockages between types:

```tcl
# hard to soft
foreach hardblk [dbGet -p top.fplan.pBlkgs.type hard] {
    dbSet top.fplan.pBlkgs.type soft
}

# hard to partial, at 20% density
dbSet [dbGet top.fplan.pBlkgs.name <blockage_name> -p].type partial
dbSet [dbGet top.fplan.pBlkgs.name <blockage_name> -p].density 20
```

Dump fences, regions, guides, and blackboxes:

```tcl
set fence_list  [concat [dbGet -p top.fPlan.groups.conType fence -e] \
                        [dbGet -p top.fplan.bndrys.type fence -e]]
set region_list [concat [dbGet -p top.fPlan.groups.conType region -e] \
                        [dbGet -p top.fplan.bndrys.type region -e]]
set guide_list  [concat [dbGet -p top.fPlan.groups.conType guide -e] \
                        [dbGet -p top.fplan.bndrys.type guide -e]]
set blackbox_list [dbGet -p2 top.insts.cell.objType ptnCell]
```

Macro obstruction information:

```tcl
dbGet -u [dbGet -p2 top.insts.cell.baseClass block].cell.allObstructions.layer.name
dbGet [dbGet -p2 top.insts.cell.baseClass block].cell.allObstructions.shapes.rect
```

{: .warning }
> Obstruction coordinates are relative to the **cell**, not the design. Use `dbTransform` to
> convert from cell coordinates to design coordinates before comparing them against anything
> else.

{: .note }
> Cadence has been migrating to a **Common UI** command set, where `get_db` replaces `dbGet`.
> Both exist in current releases. New scripts are generally better written against `get_db`;
> `dbGet` remains everywhere in existing scripts and in most online material.

---

## Selection and object queries

| Command | Purpose |
|:--|:--|
| `selectInst <name or wildcard>` | Select instances |
| `selectNet <name or wildcard>` | Select nets |
| `selectPin <name or wildcard>` | Select pins |
| `deselectNet *` | Deselect all nets |
| `zoomSelected` | Zoom to the selection |

![get commands]({{ site.baseurl }}/assets/img/innovus-commands-07.png)

The `get_*` family returns collections of design objects:

```tcl
get_pins -hier *RAM
get_lib_cells
all_fanin -to <pin>
filter_collection [get_pins -hier *RAM] -regexp {hierarchical_name == "some name"}
```

![Filtering collections]({{ site.baseurl }}/assets/img/innovus-commands-08.png)

`-hierarchical` traverses the design hierarchy, at a runtime cost. Wildcards and regular
expressions both work.

---

## Sources

- *Using dbGet Commands to Retrieve Floorplan Related Data from the Design*, Cadence Support
  — the `dbGet` recipes in this page. Cadence Support requires a login;
  the companion Common UI article is
  *Using get_db Commands to Retrieve Floorplan-Related Data from the Design*.
- *MMMC File Setup for PnR Using Innovus*, Digital System Design.
  <https://digitalsystemdesign.in/mmmc-file-setup-for-pnr-using-innovus/>
- *I/O File Setup for PnR Using Innovus*, Digital System Design.
  <https://digitalsystemdesign.in/i-o-file-setup-for-pnr-using-innovus/>
- Silicon Integration Initiative / Cadence, *LEF/DEF 5.8 Language Reference* — the object
  model the database mirrors. <http://coriolis.lip6.fr/doc/lefdef/lefdefref/LEFSyntax.html>
