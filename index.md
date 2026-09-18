---
layout: default
title: Home
nav_order: 1
description: "A beginner's guide to ASIC physical design, from transistors to GDSII."
permalink: /
---

# Physical Design Wiki

A working reference for **ASIC physical design** — the stage of chip design that turns a
gate-level netlist into a manufacturable layout.

This wiki assumes you know basic digital logic (gates, flip-flops, clocks) and nothing
else. It builds up from transistors, walks the place-and-route flow in the order a tool
actually runs it, and ends at signoff and tapeout.
{: .fs-5 .fw-300 }

[Start with the fundamentals]({{ site.baseurl }}/docs/fundamentals.html){: .btn .btn-primary }
[Jump to the flow]({{ site.baseurl }}/docs/flow/asic-design-flow.html){: .btn }
[Glossary]({{ site.baseurl }}/docs/reference/glossary.html){: .btn }

---

## How this wiki is organised

The sections are meant to be read in order the first time through. After that, use the
search box at the top of the sidebar — every page is indexed.

| Part | What it covers | Read it when |
|:--|:--|:--|
| **1. Fundamentals** | MOSFETs, CMOS logic, standard cells, macros, physical cells | You want to know *why* the physical rules exist |
| **2. The Flow & Its Inputs** | The full RTL-to-GDSII flow; the files handed to a PD engineer | You want the big picture before the details |
| **3. Timing** | STA, setup/hold, clocks and skew, variation, corners and modes | You want to understand what the tool is optimising for |
| **4. Implementation** | Floorplanning, power planning, placement, CTS, routing | You are actually running a block through the flow |
| **5. Signoff** | DRC, LVS, antenna, EM, signal integrity, power analysis, ECO | You are closing a design and preparing for tapeout |
| **6. Tools** | Cadence Innovus setup, command reference, database queries | You are sitting in front of the tool |
| **7. Study** | Glossary, interview questions, further reading | You are preparing for an interview or an exam |

---

## A note on how to read this

Physical design has a bad habit of teaching itself as a list of tool commands. The
commands change between releases and between vendors. The physics does not.

Wherever possible this wiki explains **what problem a step solves** before it shows
**how to run it**. If you only remember one thing from each page, remember the problem.

{: .note }
> Tool-specific material is confined to Part 6 and to clearly-marked "In Innovus"
> sections. Everything else should transfer to Synopsys ICC2, OpenROAD, or any other
> place-and-route tool with only vocabulary changes.

---

## Contributing

This is a Jekyll site hosted on GitHub Pages. To add or fix a page:

1. Edit the relevant Markdown file under `docs/` — there is an **Edit this page on
   GitHub** link at the bottom of every page.
2. Commit to a branch and open a pull request.
3. GitHub Pages rebuilds the site automatically once the change is merged.

New pages need a small block of front matter at the top so they appear in the sidebar:

```yaml
---
layout: default
title: Your Page Title
parent: Timing          # the section it belongs under
nav_order: 4            # position within that section
---
```

Please keep the house style: explain the problem first, keep tool commands in their own
section, and list your sources at the bottom of the page.
