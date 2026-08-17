[README.md](https://github.com/user-attachments/files/29600658/README.md)
# Claude Working Docs

Skill files, handoff documents, and implementation plans produced with Claude — the durable output of planning sessions, kept here so the thinking survives the chat threads that produced it. This repo is the source of truth; a mirror copy of everything lives in Obsidian for offline storage.

## What lives here

Four kinds of documents, each with a different job:

| Type | Naming | What it is | How it gets used |
|---|---|---|---|
| **Skill file** | `*-SKILL.md` | Instructions that teach any Claude session a repeatable workflow of mine | Upload to Claude (Settings → Capabilities/Skills). The file here is the editable master; re-upload after changes |
| **CLAUDE.md** | `*-CLAUDE.md` | Persistent context for one code project: what it is, architecture, conventions, how to work with me | Copy into that project's own repo root, where Claude Code auto-loads it. The copy here is the backup/master until the project repo exists |
| **Implementation plan** | `*-implementation-plan.md` / `*-handoff.md` | The working doc for a project: locked decisions, dead-ends, open questions, phased plan with acceptance tests | Feed it to the executing session at kickoff. It gets *updated during execution* — decisions and corrections go back into the file, then get committed here |
| **Workflow reference** | descriptive name | Step-by-step technical procedure (stable once proven) | Follow it; revise with a Rev log entry when a run surfaces corrections |

## Current inventory

| File | Type | Status |
|---|---|---|
| `claude-usage-roadmap.md` | Roadmap | Reference — the July 2026 review that produced most of the docs below |
| `project-handoff-SKILL.md` | Skill | **Active** — upload to Claude; governs how all future handoff docs get made |
| `arch-spec-research-SKILL.md` | Skill | **Active** — spec research for SAI work; validate on first real spec question |
| `pi-consolidation-handoff.md` | Handoff/plan | Awaiting execution — gated on the Option A/B decision + hardware confirmation |
| `revit-tool-CLAUDE.md` | CLAUDE.md | Awaiting execution — copy to the kitchen-equipment tool repo when created |
| `revit-tool-implementation-plan.md` | Implementation plan | Awaiting execution — Phase 1 gated on locating `script.py` |
| `distant-terrain-workflow-rev3.md` | Workflow reference | **Active, Rev 3** — Route A (Revit points-CSV toposolid) + Route B (D5 heightmap) off one QGIS pipeline. Supersedes `distant-terrain-workflow(1).md` (Rev 2, retained for history) |
| `very-distant-terrain-workflow.md` | Workflow reference | **Active, Rev 1.1** — standalone Revit/Enscape guide for terrain >15 km (10+ mi); midpoint-rebasing trick + optional viewshed cull |
| `d5-terrain-checklist.md` | Workflow reference | **Active, Rev 1** — standalone D5-only (Route B) heightmap checklist, extended for ~20-mile views |
| `terrain-handoff.md` | Handoff/plan | **Active, Rev 2** — companion decision record; renderer decision rule + rejected alternatives |

## Conventions

- **Docs are living during execution.** When a build session corrects course, the correction gets written into the plan (locked decisions, dead-ends table) in that same session, then committed. A plan that drifted from reality is worse than no plan.
- **Rev logs on revised docs.** Substantive revisions get a dated entry noting what changed and why (see `terrain-handoff.md` for the pattern).
- **Dead-ends never get deleted.** Rejected approaches stay in the doc with their rejection reason and a "don't retry unless" condition — that's the record that stops a future session from resurrecting them.
- **No secrets, ever.** Docs describe where credentials live and how to create them; actual tokens, keys, and secret URLs (e.g., iCal addresses) never get committed.
- **Obsidian mirror.** After committing new or revised docs here, copy them to Obsidian.

## Kicking off an execution session

The short version of what these docs are for:

1. Give the session the relevant plan (plus the CLAUDE.md, for code projects).
2. Say: *"Read this, then interview me on the open questions before doing anything."*
3. Build phase by phase; check the acceptance tests at each gate.
4. Make sure corrections land back in the doc before the session ends — then commit and mirror.

## Adding future projects

New project → new handoff/plan doc, produced with the `project-handoff` skill (which encodes the format: goal, status, locked decisions with rationale, dead-ends, open questions, riskiest-unknown-first phases, acceptance tests). If this repo starts feeling crowded, the natural split is `skills/` at top level and a folder per project — but flat is fine until it isn't.
