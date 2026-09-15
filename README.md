# Physical Design Wiki

A beginner-oriented reference for ASIC physical design, built as a GitHub Pages site.

Source notes were reorganised into a structured wiki: transistors → the flow → timing →
implementation → signoff → tools → study material. Every page ends with a list of the
public sources its explanations were checked against.

---

## Deploying this to GitHub Pages

1. **Create a repository** (e.g. `pd-wiki`) and push the contents of this folder to the
   `main` branch.

2. **Edit `_config.yml`.** Three fields need your values:

   ```yaml
   url: "https://YOUR-USERNAME.github.io"
   baseurl: "/pd-wiki"                       # "" if using a user/org site
   gh_edit_repository: "https://github.com/YOUR-USERNAME/pd-wiki"
   ```

   Also update `nav_external_links` to point at your repo.

3. **Turn on Pages.** Repository → Settings → Pages → Source: *Deploy from a branch* →
   Branch: `main`, folder `/ (root)`. Save.

4. Wait a minute or two. The site appears at `https://YOUR-USERNAME.github.io/pd-wiki`.
   Shorten that with TinyURL or similar if you want something memorable to hand out.

No build tooling is required locally — GitHub builds the site. The theme
([just-the-docs](https://just-the-docs.com)) is pulled in via `remote_theme`, which
GitHub Pages supports natively, and it provides the sidebar navigation and the full-text
search box at no extra effort.

### Optional: previewing locally

```bash
bundle init
bundle add jekyll jekyll-remote-theme jekyll-seo-tag
bundle exec jekyll serve
```

Then open <http://localhost:4000>.

---

## Repository layout

```
.
├── _config.yml              site configuration, theme, search, nav
├── index.md                 home page
├── assets/img/              243 figures, named <topic>-NN.png
├── image-manifest.json      maps every figure back to its source note
└── docs/
    ├── fundamentals/        Part 1 — transistors, CMOS, cells, sites, rows
    ├── flow/                Part 2 — the ASIC flow and its input files
    ├── timing/              Part 3 — STA, clocks, variation
    ├── implementation/      Part 4 — floorplan, power, placement, CTS, routing
    ├── signoff/             Part 5 — verification, SI, power, ECO
    ├── tools/               Part 6 — Innovus setup and commands
    └── reference/           Part 7 — glossary, interview questions, reading
```

## Status

All seven parts are written. 30 pages, 243 figures, every internal link and anchor verified.

| Part | Pages |
|:--|:--|
| 1. Fundamentals | MOSFETs and CMOS logic; cells, macros, sites, rows, tracks, utilisation |
| 2. The Flow and Its Inputs | The ASIC design flow; design inputs and file formats |
| 3. Timing | STA basics; clocks, skew and latency; variation, corners and analysis modes |
| 4. Implementation | Floorplanning; power planning; placement; CTS; routing |
| 5. Signoff | Physical verification; signal integrity; power and rail analysis; timing closure and ECO |
| 6. Tools | Innovus setup and design import; Innovus command reference |
| 7. Study and Reference | Glossary; interview questions; learning plan; open questions |

---

## Adding a page

Create a Markdown file in the right section folder with front matter:

```yaml
---
layout: default
title: Your Page Title
parent: 3. Timing        # must match the section's `title` exactly
nav_order: 4             # position within the section
---
```

House style, to keep the wiki readable for someone new:

- **Explain the problem before the command.** Tool syntax dates; the physics does not.
- **Keep tool-specific material in marked sections** so the rest transfers to other tools.
- **Cite at the bottom.** List the public sources you checked the explanation against.
- Use the callout styles defined in `_config.yml`: `.note`, `.tip`, `.warning`,
  `.interview`.

---

## A note on the figures

`assets/img/` contains screenshots carried over from the original study notes, including
material that appears to come from vendor training courses. They were kept because they
carry information the text does not, but **check the licensing before making this
repository public.** If in doubt, keep the repo private or unlisted, or replace the
vendor-sourced figures with redrawn originals.

`image-manifest.json` records which source note each figure came from, which makes an audit
or a replacement pass straightforward.
