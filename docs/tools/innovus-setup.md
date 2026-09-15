---
layout: default
title: Innovus Setup and Design Import
parent: 6. Tools
nav_order: 1
---

# Innovus Setup and Design Import
{: .no_toc }

<details open markdown="block">
  <summary>On this page</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## Starting up

![Innovus startup]({{ site.baseurl }}/assets/img/cadence-innovus-block-implementation-certification-01.png)

Launch an `xterm` alongside the tool to act as a console for messages — the GUI's own
message area is easy to lose track of, and the shell output is where the useful information
usually is.

Cadence ships a library of Tcl scripts for automating common tasks, under the **gift**
directory:

```
<installation_directory>/INNOVUS***/<build_version>/<binary>/share/innovus/gift/scripts/tcl
```

Worth browsing before writing your own version of something that already exists.

### `enc.tcl`

For design-specific settings, create an `enc.tcl` in the working directory. Tcl variables
set there are applied every time Innovus is opened from that directory, so per-design
preferences do not have to be re-entered.

![enc.tcl]({{ site.baseurl }}/assets/img/cadence-innovus-block-implementation-certification-02.png)

### Running a script

Launch the tool, then source your flow script:

```tcl
innovus
source <source_file_name>.tcl
```

![Sourcing a script]({{ site.baseurl }}/assets/img/cadence-innovus-block-implementation-certification-03.png)

{: .tip }
> Scripting the flow is not optional in practice. You will rerun the same block many times,
> and a flow you can reproduce from a script is the difference between a reproducible result
> and a result nobody can explain a week later.

---

## Importing a design

The collateral Innovus needs is the list from
[design inputs]({{ site.baseurl }}/docs/flow/design-inputs/): netlist, SDC, LEF, Liberty,
technology and extraction files, and optionally a DEF and an I/O file.

![Innovus inputs]({{ site.baseurl }}/assets/img/cadence-innovus-block-implementation-certification-04.png)

![Input collateral]({{ site.baseurl }}/assets/img/cadence-innovus-block-implementation-certification-05.png)

### The design import window and `design.globals`

You can fill in the collateral manually in the GUI's design import window. Having done so,
**save it all into a `design.globals` file** — a set of Tcl commands recording the paths and
settings for the design.

![Design import]({{ site.baseurl }}/assets/img/cadence-innovus-block-implementation-certification-06.png)

From then on, loading the design is two commands:

```tcl
source design.globals
init_design
```

![design.globals contents]({{ site.baseurl }}/assets/img/cadence-innovus-block-implementation-certification-07.png)

{: .tip }
> Do the GUI import **once**, save `design.globals`, and never use the import window again.
> The file is also the artefact to check first when something is mysteriously wrong — a
> wrong library path or a stale SDC is immediately visible there, where it is not in a GUI
> dialog nobody has opened in three weeks.

### Sanity checks after import

```tcl
checkDesign -netlist
```

Run before floorplanning. It confirms every input was read correctly and every reference
resolved. A missing library caught here costs a minute; caught at routing it costs days.

---

## GUI shortcuts

| Key | Action |
|:--|:--|
| `f` | Fit the floorplan to the view |
| `shift-z` | Zoom out |
| `e` | Open the Wire Editor |

`zoomSelected` (or `gui-zoom -selected`) zooms to whatever is currently selected — the
fastest way to find a cell or net you have located by name.

The number of currently selected elements is displayed at the bottom right of the GUI, which
is a useful confirmation that a wildcard selection matched what you expected.

---

## Sources

- Cadence Innovus documentation and the Innovus Block Implementation training materials, as
  reflected in the figures above. Command syntax and available options change between
  releases — check `<command> -help` in the shell for the version you are running.
- *MMMC File Setup for PnR Using Innovus*, Digital System Design — collateral required at
  design import and the order it is read.
  <https://digitalsystemdesign.in/mmmc-file-setup-for-pnr-using-innovus/>
- *I/O File Setup for PnR Using Innovus*, Digital System Design — I/O assignment file
  handling during import. <https://digitalsystemdesign.in/i-o-file-setup-for-pnr-using-innovus/>
