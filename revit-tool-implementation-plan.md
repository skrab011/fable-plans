# Kitchen Equipment BIM Tool — Implementation Plan
Prepared: July 1, 2026 · Source: uploaded code (scraper.py, config.py, SETUP.md) + project history · Executor: Claude Code (or desktop chat with file access) on Jacob's work machine

## Goal
Finish the pivot to a self-contained pyRevit extension: button click → paste URLs → flexed `.rfa` families in a project folder, reliable enough to hand to SAI coworkers. Architecture and conventions live in `CLAUDE.md`; this doc tracks what's left.

## Current status
- **scraper.py: complete and structured well.** Two-store support via the `SITE_ROUTERS` adapter pattern, generic `xray_find_spec` fallback, Machete title cleaner, architectural dimension parser, CAPTCHA-tolerant headful browsing, stdin/manual dual entry points, CSV output.
- **config.py: complete.** Template paths, plumbing-keyword routing, CSV contract, Revit parameter names. Paths are placeholders (`C:\Path\To\Your\...`) — real office paths not yet set. Note: the uploaded `config.pdf` extraction shows a garbled `TEMP_CSV_PATH` line; the `.py` file is the source of truth.
- **SETUP.md: complete** — describes install, testing, troubleshooting, and how to add a store.
- **script.py: `OPEN` — not provided.** SETUP.md describes it as a finished pyRevit button (URL dialog → subprocess scrape → folder picker → family generation → summary dialog), but the file wasn't uploaded and the project stalled "when the workweek ended." Its true state — complete / partial / described-but-unwritten — is the project's biggest unknown and drives Phase 1.
- **No Git repo.** Files live in a pyRevit pushbutton folder. (Open Question 6.)
- **Superseded:** the standalone terminal tool and the GUI `.exe` shell from the Gemini phase. No clean handoff doc exists from that work; the uploaded files are the surviving artifact of record.

## Locked decisions
| Decision | Rationale |
|---|---|
| All-in-one pyRevit extension (not standalone GUI/.exe) | One-click workflow inside Revit; simplest thing to share with coworkers |
| CPython subprocess for scraping; CSV as the bridge | Playwright cannot run under Revit's IronPython; the CSV decouples the two failure domains |
| AI extraction deferred pending SAI's AI-usage policy | Don't build what can't be deployed; adapter pattern means it slots in later as one more adapter with zero rework |
| `headless=False` + human CAPTCHA click | Reliable against bot detection; cookie persists for the session |
| Parse failures → `0.0000` → generator skips parameter | A family with one missing dimension beats a crashed batch |
| No voltage/wattage fields | Tool serves architectural space planning, not MEP |
| Folder picker at runtime for output | Families are project-specific; office server paths vary per project |

## Dead-ends
| Tried | Why it failed / was dropped | Don't retry unless |
|---|---|---|
| Standalone terminal scraper | Worked, but usable only by Jacob — not shareable | Never (superseded, not broken) |
| GUI shell packaged as .exe | Superseded by the pyRevit-button decision; extra distribution surface for no benefit | Non-Revit users at SAI need the scraper alone |
| Running Playwright under IronPython | Incompatible — hence the subprocess bridge | Never |
| Per-store scraping generalized via one mega-function | Replaced by SITE_ROUTERS adapters + xray fallback | — (current pattern is the fix) |

## Open questions (ask Jacob before building)
1. **script.py — what actually exists?** Upload it if it exists in any form. If it's partial, where did it stop? If SETUP.md was written ahead of the code (docs-as-spec), say so — Phase 1 becomes a build instead of a verification.
2. **Has any URL ever gone end-to-end** (paste → scrape → family opens and flexes in Revit)? Determines whether Phase 1 is debugging or first integration.
3. **Do both FlexBox `.rft` templates exist** with the required parameters (`Width/Depth/Height` as Length, `Finish`, `Notes`), and do they flex correctly when values are typed manually in the family editor? (Template problems masquerade as script problems — rule them out first.)
4. **Which Revit version(s) does SAI run?** Affects pyRevit compatibility and any API differences.
5. **Jacob homework:** fill in the two real template paths in `config.py` on the work machine.
6. **Repo?** Recommendation: create a private `skrab011/kitchen-equipment-bim` repo so iteration happens in Claude Code with real diffs instead of full-file rewrites. Confirm whether work-machine Git access exists; if not, the pushbutton folder + manual backups is the fallback.
7. **Deployment model for coworkers** (later phase, but shapes decisions): shared network extension folder that everyone's pyRevit points to, or per-machine installs? Each coworker's machine also needs CPython + Playwright — is that install acceptable office-wide?

## Phased plan
### Phase 1 — Prove the bridge end-to-end (riskiest unknown)
Resolve script.py's state (OQ 1–3), then get **one URL → one flexed family** through the full path: button click, dialog, subprocess scrape, CSV read, template routing, parameter injection with inches→feet conversion, save to picked folder, summary dialog. Build or repair only what this single-URL path needs.
**Acceptance tests:** clicking the button on one WebstaurantStore URL produces an `.rfa` in the chosen folder; opening it in the family editor shows correct Width/Depth/Height (flexing when changed), Manufacturer/Model/URL populated; the summary dialog reports 1 success, 0 errors.

### Phase 2 — Batch hardening
Multi-URL runs with mixed outcomes: unsupported domains, timeouts, 0.0 dimensions, duplicate equipment names (name-collision handling for the `.rfa` filenames). Summary dialog must account for every input URL — created / skipped / failed, with reasons.
**Acceptance tests:** a 6-URL batch containing one unsupported domain, one bad URL, and one page with unparseable dimensions produces the right families for the rest and a summary that itemizes all six outcomes; a repeated equipment name doesn't silently overwrite an earlier family.

### Phase 3 — Formalize the adapter contract + third store
Write the adapter contract into scraper.py as a documented docstring/template function ("copy this, fill in selectors"), then implement `acitydiscount.com` (or Jacob's preferred third store) against it, per SETUP.md's existing recipe.
**Acceptance tests:** third store scrapes to a correct CSV row standalone (`python scraper.py`); a new-store checklist exists that a future session can follow without reading the whole file.

### Phase 4 — Office deployment
Per OQ 7: package the extension for coworkers, write a coworker-facing one-page install/use guide (simpler than SETUP.md — they configure nothing), and run one coworker pilot.
**Acceptance tests:** a coworker generates a family on their own machine without Jacob touching the keyboard; config paths resolve on their machine.

### Phase 5 — Deferred: AI extraction adapter
Blocked on SAI's AI-usage policy. When unblocked: one adapter that takes any URL, sends page content to a model, returns the standard dict validated against the CSV contract. Slots into `SITE_ROUTERS` as a fallback for unsupported domains. Do not start before Jacob confirms the policy question is settled.

## Failure-mode defaults
- **Per-URL failures: silent-with-log during the run, loud in the summary.** One bad URL never aborts the batch; every URL's fate appears in the closing dialog.
- **Dimension parse failure: `0.0000` in CSV; generator skips that parameter** and the summary flags the family as "created with missing dimensions" — never a crash, never a silently wrong dimension.
- **Missing template / bad config path: fail loud immediately**, before any scraping — config errors are setup problems and should stop the run at step zero.

## Resources
- Uploaded artifacts of record: `scraper.py`, `config.py`, `SETUP.md`
- Playwright install per SETUP.md (CPython 3.10–3.12, `pip install playwright`, `python -m playwright install chromium`)
- pyRevit already installed on the work machine
- No API keys or credentials of any kind in this project's current scope

## Working with Jacob
Interview first — Open Questions 1–5 gate Phase 1. Plain-language walkthroughs alongside commented code (coding novice, expert architect). Plan Mode before builds. Push back with reasoning. Record every correction and decision back into this file in the same session.
