---
layout: default
title: Open Questions
parent: 7. Study and Reference
nav_order: 4
---

# Open Questions

Things this wiki does not answer well, or answers only partially. If you know one of these
properly, the wiki would benefit from it —
see [Contributing]({{ site.baseurl }}/#contributing).

---

## Answered, but worth deeper treatment

**What are the consequences of an incomplete MMMC view file?**

Partially answered in
[variation and analysis modes]({{ site.baseurl }}/docs/timing/variation/#corners-modes-and-mmmc):
the failure is silent. An omitted corner is simply never checked, so a violation that only
appears under those conditions passes implementation and surfaces at signoff or in silicon.
It does not produce less accurate results so much as *absent* results.

Still open: what a realistic minimum set of views looks like for a production block, and how
teams decide which corner combinations can safely be dropped for runtime.

**Is the LEF something we can customise, or is it part of the standard cell library?**

Answered in [design inputs]({{ site.baseurl }}/docs/flow/design-inputs/#lef-library-exchange-format):
both, depending on which LEF. The **technology LEF** comes from the foundry and encodes
manufacturing rules — treat it as read-only. The **cell library LEF** normally ships with the
standard cell library, but if a block lacks one (new macro, hard IP delivered without an
abstract), you generate an abstract view from the layout.

Still open: practical guidance on abstract generation quality — what a bad auto-generated
abstract looks like and how it bites you later.

---

## Genuinely open

- **Clock cell selection.** Which cells the tool will use in a clock tree is governed by
  library setup and CCOpt properties rather than a universal rule. Worth documenting how to
  inspect and control the permitted list for a given flow.
- **Corner reduction in practice.** How production teams decide which of the combinatorially
  many mode/corner pairings actually need analysing.
- **Dynamic IR drop methodology.** This wiki covers static rail analysis reasonably and
  dynamic only in outline.
- **Multi-voltage and power domain implementation.** CPF/UPF is mentioned but the
  implementation flow — level shifters, isolation cells, retention flops, power switch
  placement — is not covered.
- **Hierarchical and partition-based flows.** Everything here assumes a flat block.
- **3D-IC and advanced packaging.** Entirely out of scope at present.
