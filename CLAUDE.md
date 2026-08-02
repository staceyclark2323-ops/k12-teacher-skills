# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

`k12-teacher-skills` is Anthropic's open-source repo for the agent Skills bundled with **Claude for Teachers**. It packages a Claude Code / Claude plugin marketplace, not a conventional application: there's no server, build step, or long-running process. The deliverable is a `.claude-plugin` marketplace containing two Agent Skills that turn Claude into a lesson-authoring assistant for K-12 teachers:

- **`k12-lesson-planning`** — builds a standards-aligned lesson plan, student-facing materials, and a teacher observation template from scratch (math, ELA, science, social studies).
- **`k12-lesson-differentiation`** — takes an existing lesson and tiers it into below/at/above-grade-level versions.

Both skills are co-developed with **Learning Commons** (see `NOTICE`) and can optionally ground their output in the Learning Commons Knowledge Graph MCP connector, but work without it too. The repo is Apache-2.0 licensed (`LICENSE`), maintained by Anthropic, PBC (`CODEOWNERS`: `@hud-ant`, `@akshit-ant`), and contributions require signing the CLA (`CLA.md`) via the bot-driven flow in `.github/workflows/cla.yaml`.

There is no traditional install/build/run/test workflow — read the "Working with this repo" section below before assuming otherwise.

## Repository layout

```
plugin/                              the actual Claude Code plugin (this is what ships)
  .claude-plugin/
    marketplace.json                 marketplace metadata: one plugin entry, "k12-teacher-skills"
    plugin.json                      plugin manifest (name/version/description/author)
  .mcp.json                          declares 9 optional third-party MCP servers teachers can enable
                                      (ASSISTments, Brisk Teaching, Canva, Coteach, Diffit, Eedi,
                                      MagicSchool, Snorkl, TeachFX)
  skills/
    k12-lesson-planning/             Skill 1
    k12-lesson-differentiation/      Skill 2

evals/                               rubric-based LLM-as-judge eval framework (not code tests)
  k12-lesson-planning/rubrics/*.csv
  k12-lesson-differentiation/rubrics/*.csv

.github/workflows/cla.yaml           the only CI: CLA signature verification
README.md, CONTRIBUTING.md, CODE_OF_CONDUCT.md, SECURITY.md, NOTICE, CLA.md, CODEOWNERS, LICENSE
```

There is no `package.json`, `pyproject.toml`, `requirements.txt`, or any other dependency manifest anywhere in the repo, and no linter/formatter config. The only generated artifacts ignored are Python bytecode and editor/Claude local state (`.gitignore`: `.DS_Store`, `__pycache__/`, `.claude/`, `.idea/`, `.vscode/`).

### Anatomy of a skill (`plugin/skills/<skill-name>/`)

Both skills share the same shape:

- **`SKILL.md`** — the skill itself: YAML frontmatter (`name`, `description`, `license`) followed by a long, carefully written instruction document. The `description` field is what Claude uses to decide *when* to trigger the skill, so it's written defensively — it states what the skill is for, what it is explicitly **not** for, and how to disambiguate from the sibling skill (e.g. a new-lesson request that also asks for differentiated materials stays inside `k12-lesson-planning`; it does not hand off to `k12-lesson-differentiation`). Body content carries an SPDX header (`SPDX-FileCopyrightText: Anthropic, PBC` + `Learning Commons`, `SPDX-License-Identifier: Apache-2.0`).
- **`references/`** — subject-specific pedagogy guidance the skill loads as needed: `math.md`, `ela.md`, `science.md`, `social_studies.md`, plus `learning-commons-kg.md` (how to use the Knowledge Graph connector when present) and an example material-source JSON (`example_lesson.json` / `example_differentiation.json`) documenting the schema the skill authors into.
- **`scripts/`** — the deterministic rendering pipeline that turns the JSON Claude authors into teacher deliverables:
  - `render_all.sh` — the entrypoint (`bash scripts/render_all.sh lesson.json "$OUTPUT_DIR"`). Ensures `python-docx==1.1.2` is installed (pip-installs it on demand), calls `render_documents.py` to produce both `.docx` (the real deliverable) and `.html` (a preview twin that renders even without `python-docx`), copies the source JSON alongside the output so later revision turns can re-render from it, and fails loudly if `.docx` output couldn't be produced. It also mirrors output into `$OUTPUT_DIR` if the render happened elsewhere first.
  - `render_documents.py` — CLI that dispatches to the docx/html renderers for every entry in the JSON's `documents[]` array.
  - `render_lesson_docx.py` / `render_lesson_html.py` — the actual document builders (docx via `python-docx`, html as a lightweight twin), sharing helpers from `lesson_common.py` (block model, print-safety rules).
  - `theme.css` — styling for the HTML twin.
  - The two skills' `scripts/` directories are near-duplicates of each other, diverging only where subject-specific rendering logic requires it — when fixing a bug in one, check whether the same bug exists in the sibling skill's copy.
- **`LICENSE`**, **`references/NOTICE`** — per-skill copies of licensing/attribution.

## Working with this repo

There is no build, lint, or automated test suite. Language is plain Python 3 (stdlib + `python-docx`, installed ad hoc by `render_all.sh`) and bash for the entrypoint script.

- **To exercise a skill end-to-end**: load it into Claude (Claude Code or claude.ai) and drive it conversationally per its `SKILL.md` — per `CONTRIBUTING.md`, this *is* the testing method for this repo.
- **To exercise just the rendering pipeline** without going through the LLM: hand-author or copy an example JSON (`references/example_lesson.json` or `references/example_differentiation.json`) and run `bash plugin/skills/<skill-name>/scripts/render_all.sh <path-to-json> <output-dir>`.
- **Evals** (`evals/`) are rubric CSVs, not an automated harness: generate materials with a skill, then feed the materials plus the relevant rubric CSV(s) (columns: `ID`, `Bucket` [P=Pedagogy, R=Rigor, O=Output, M=Model Scaffolding], `Criterion`, `What pass requires`, `Notes`, `Conditional`) and the judge system prompt described in `evals/README.md` to a model acting as an LLM judge, which returns pass/fail per criterion.
- **CI**: `.github/workflows/cla.yaml` only checks CLA signatures on PRs (via a sha-pinned fork of `cla-github-action` plus a custom `gh api` check for merge-queue PRs). Nothing else runs in CI — there is no lint/test/build gate to satisfy before merging.

## Conventions

- **Skill descriptions are load-bearing**: when editing a `SKILL.md` frontmatter `description`, preserve the explicit triggers, explicit "do NOT load for..." exclusions, and the disambiguation language between the two skills — these aren't incidental prose, they're what routes requests to the right skill (or to neither).
- **Keep the two skills' `scripts/` directories in sync**: they're intentionally near-duplicated; a fix to `lesson_common.py`, `render_lesson_docx.py`, etc. in one skill's directory almost always needs the matching fix in the other's, unless the divergence is subject-specific by design.
- **Licensing headers**: new `SKILL.md` / reference files should carry the same SPDX header pattern (`SPDX-FileCopyrightText: Anthropic, PBC` + `Learning Commons`, `SPDX-License-Identifier: Apache-2.0`) used throughout.
- **Branching/PRs**: per `CONTRIBUTING.md`, fork and branch from `main`; PRs need at least one maintainer review and a signed CLA (enforced by CI). No specific commit-message or branch-naming scheme is documented — the repo's git history is a single initial commit, so there's no established convention to follow beyond clear, descriptive messages.
- **Versioning**: `plugin/.claude-plugin/plugin.json` and `plugin/.claude-plugin/marketplace.json` both carry the plugin version (currently `0.6.0`) — keep these two in sync when bumping.
