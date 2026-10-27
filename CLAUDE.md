# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Public documentation site for the CCI (Commonwealth Cyber Initiative) xG Testbed, built with Sphinx + reStructuredText and the `sphinx_rtd_theme`. There is no application code, test suite, or linter. "Testing" a change means building the site and checking for Sphinx warnings.

Pushes to `main` on GitHub (`cci5g-Admin/CCI_TESTBED_DOCS_PUBLIC`) trigger a ReadTheDocs build via `.readthedocs.yaml` (Ubuntu 22.04, Python 3.12, apt `graphviz`), published at https://cci-testbed-docs.readthedocs.io/. The handover guide (`CCI_xG_Testbed_Documentation_Guide.docx/.pdf`) and the boilerplate `README.md` still refer to GitLab (`gitlab.com/ccixgtestbed/cci_docs`), but GitHub is the current remote.

## Commands

```bash
pip install -r requirements.txt              # sphinx, sphinx_rtd_theme, sphinx-autobuild, sphinx-copybutton
sphinx-autobuild source build/html           # live-reload preview at http://127.0.0.1:8000
sphinx-build -b html source build/html       # one-off build (what ReadTheDocs runs); check output for warnings
```

`build/` is gitignored.

## Structure and conventions

- **Navigation lives entirely in `source/index.rst`.** It is a series of captioned `toctree` blocks, one per sidebar section. A new page appears in the site only after it's added to a toctree there. Many `toctree`s are commented out with `.. ..`. Their pages (top-level `4g-deployment.rst`, `5g-deployment.rst`, `oran.rst`, `opensas/`, `net_plan_config.rst`, etc.) are legacy and not published. Likewise, `getting_started/openstack.rst`, `openstack_v1.rst` and `openstack_v2.rst` are superseded by `openstack_v3.rst`, and `sample_experiments/near-rtOld.rst` by `near-rt.rst`. Check `index.rst` before editing a page, to make sure you're editing the version that's actually live.
- **Live content areas:** `overview/`, `sites/`, `getting_started/`, `software_architecture/{oran,opensas}/`, `sample_experiments/`, plus top-level `otic.rst`, `badges.rst`, `team_info.rst` (team roster and alumni), `research_area.rst` (publications), `partnerships.rst`, `help.rst`, `faqs.rst`, `contact_us.rst`.
- **Static assets** go in `source/_static/` (some older images are in `images-4g/`, `images-5g/`, `images-oran/`). Image paths are relative to the `.rst` file.
- **Videos** are embedded with `.. raw:: html` `<video>` tags, not a Sphinx directive. The `src` path must match the page's depth: `_static/x.mp4` from a top-level page, `../_static/x.mp4` from a page in a subdirectory. Videos are stored as normal git files (Git LFS was dropped), so each must stay under GitHub's 100 MB file limit. Compress large ones before committing.

## `conf.py` quirks

- Only `custom.css` is loaded. `_static/width.css` is intentionally not loaded: it would add a 10px black border to every figure (including team photos). Full-width content comes from `_templates/layout.html`, which overrides `.wy-nav-content` max-width.
- `graphviz_dot` is unset, so Sphinx uses `dot` from PATH. Locally, install it with `brew install graphviz` before using a `graphviz` directive. The only one today (in `team_info.rst`) is commented out.
- `source/conf.py` uses CRLF line endings; keep them when editing.
- When adding a Sphinx extension, add it to both `extensions` in `conf.py` and `requirements.txt`, or the ReadTheDocs build will fail.
