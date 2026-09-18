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
