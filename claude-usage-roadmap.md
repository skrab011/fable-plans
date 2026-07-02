# Claude Usage Roadmap — Skills & Project Handoff Plan
**Prepared by:** Claude Fable 5 · **Date:** July 1, 2026
**Purpose:** Handoff document for Opus/Sonnet sessions. Reviewed conversation history across work (SAI) and personal projects to identify (1) skills worth building and (2) existing projects where a focused re-pass would produce meaningfully better results.

---

## Part 1: Skills to Develop (Ranked)

These are ranked by leverage — how much each skill improves *everything else* you do with Claude, not just one task. Each can be built with any capable model using the skill-creator workflow (draft → test on real prompts → iterate → package as a `.skill` file).

### Skill 1: Project Handoff Brief (`project-handoff`)
**Why first:** You've independently invented this pattern three times — the Google Home alarm handoff doc, the weather dashboard P1–P8 prompt sequence, and the ballot research → quiz pipeline. It's your single most effective workflow, and it's currently ad hoc. Codifying it means any model produces a handoff doc at the quality of your best one, every time.

**What it captures:**
- Standard sections: project goal, locked decisions (with rationale), dead-ends table (what was tried and why it failed — the alarm doc proved how valuable this is), open questions for the implementing model, and acceptance tests per phase
- The "interview me first" instruction pattern you've adopted
- Phase ordering rule: front-load the riskiest unknown (e.g., the alarm plan's bare-bones cast test before any calendar logic)
- Fail-loud vs. fail-silent defaults stated explicitly
- Your standing preferences baked in: plain-language code explanations, Plan Mode before big builds, record corrections back into the doc

**Trigger phrases:** "handoff doc," "prep this for Claude Code," "package this plan," end-of-research-session wrap-ups.

### Skill 2: Spec & Code Research (`arch-spec-research`)
**Why second:** The Gunnison County roof underlayment session is a template for a recurring work task: evaluate a spec question against manufacturer requirements, model code provisions, and firm standards, then produce drawing-ready specification language. That workflow has repeatable structure worth locking in.

**What it captures:**
- Source hierarchy: manufacturer installation manuals/warranty requirements → IRC/IBC (with Climate Zone context — you're mostly CZ 6/7 work) → firm standards (self-adhered membranes, high-heat underlayment under metal, Grace I&W as house standard)
- Manufacturer roster: Sheffield Metals, Englert, Metal Sales (extendable)
- Output format: findings summary → recommended spec language (copy-paste ready for drawings) → flags/items to confirm (e.g., "confirm panel profile before cross-referencing the install manual")
- Rule: cite the actual document (manual section, code provision) rather than general claims; flag confidence level on anything not directly verified

**Trigger phrases:** spec questions, "what does the manufacturer require," code-compliance checks, assembly reviews.

### Skill 3: Mobile PWA Game Dev (`pwa-game-dev`)
**Why third:** Vortex and Shui Pái established a mature, repeatable stack that lives only in old conversation threads. Without a skill, every new game (or major update) re-derives these lessons — some the hard way, like the React artifact-viewer failure that forced the Vortex rebuild.

**What it captures:**
- Architecture: single-file HTML, vanilla JS, no build tooling, GitHub Pages deploy, plus the standard companion files (manifest.json for fullscreen portrait, sw.js with versioned cache-first serving, README, inline SVG icon)
- iPhone 15 Pro Max portrait optimizations: `dvh` units, `visualViewport` listeners, `env(safe-area-inset)` padding, `touch-action: manipulation`
- Canvas games: virtual coordinate space (390×844) scaled to viewport with hit-testing in virtual coords — this specifically solved the input-mismatch problem
- Interaction rules from Shui Pái iterations: fire-and-forget animations that never block input, immediate tap response, localStorage for progress/persistence, seeded RNG for reproducible levels
- Your aesthetic defaults: dark-mode-first, minimal chrome, no text labels where an icon works

**Trigger phrases:** any new game idea, updates to Vortex or Shui Pái, "mobile game," "GitHub Pages game."

### Skill 4: Deep-Dive Buyer's Guide (`research-buyers-guide`)
**Why fourth:** You run this pattern constantly — air quality sensors, LiDAR apps, laptop replacement value, Pi hardware. The individual outputs have been good, but their structure varies session to session. A skill standardizes the parts that made the best ones work.

**What it captures:**
- Always open with 2–3 clarifying questions (primary use case, hard vs. soft requirements, budget posture) before searching
- Output structure: conceptual framing of the problem first (e.g., infiltration ratio for air sensors) → staged recommendation (cheap/leveraged first step, then the fuller solution) → product-by-product comparison → hard-requirement disqualifications listed explicitly (like the cloud-processing exclusions in the LiDAR guide)
- Integration lens: check Home Assistant compatibility, local-processing options, and API access by default — these keep mattering to you
- Flag advocacy-sourced figures and pricing volatility; include fallback positions where relevant (the insurance guide's staged negotiation positions were a highlight worth repeating)

### Skill 5: Email Proofreading v3 (upgrade existing skill)
**Why fifth:** v2.0 is solid and built on real data (37 emails). The upgrade isn't content — it's validation. Run it through the skill-creator eval loop: test it against a fresh batch of sent emails it hasn't seen, check whether the redlines match what you'd actually accept, and tune the trigger description so it fires reliably. Two additions worth considering while you're in there: a register for client-facing *documents* beyond email (proposal cover letters, ASI narratives), and a rule set for reply-thread context (proofing a reply mid-thread vs. a cold email).

---

## Part 2: Projects Worth a Re-Pass (Ranked)

Ranked by a blend of value, current momentum, and how much a better approach changes the outcome. For each: what exists, why a re-pass wins, and what to hand the next model.

### #1 — Weather Dashboard PWA (finish it)
**Status:** Strong planning phase complete (locked spec, CLAUDE.md, P1–P8 prompts, repo live with PAT auth resolved). Build not meaningfully started.
**Why it's first:** Highest personal value (daily use, wildfire season is *now*), and all the hard thinking is done — it's pure execution debt. It's also your best Claude Code training ground.
**Where a re-pass improves the original plan:**
- **Consolidate the planning docs.** Four markdown files + P1–P8 was right for June-you; current Claude Code works better from one tight CLAUDE.md plus a single phased build plan with acceptance tests per phase (the alarm-plan format). The P3a–P3d per-source split is still the right idea — keep that.
- **Front-load the riskiest source.** The CAIC feed is undocumented and could break the plan; build and validate that fetcher first (with failure isolation), not in spec order.
- **Add a stub for the indoor air sensor.** The air-quality research (AirGradient Open Air / IKEA VINDSTYRKA + ESPHome) will eventually feed an indoor/outdoor infiltration ratio card. Don't build it yet — but design the PM2.5 card so a second data series drops in without a refactor.
**Handoff:** Point the model at the repo, have it read all existing planning docs, consolidate into CLAUDE.md + one phased plan, then build phase by phase with plain-language explanations. Interview you only where the docs are silent.

### #2 — Revit Family Generator (rebuild the scraper, finish the PyRevit extension)
**Status:** Stalled mid-pivot. Built with Gemini: working scraper for one store → CSV → PyRevit family generation, plus a GUI shell. The move to a fully self-contained PyRevit extension was in progress when it stalled.
**Why it's second:** Real work value (shareable with SAI coworkers), and it's the project where a different approach most changes the outcome — not just resumes it.
**Why Claude can do better:**
- **The multi-store scraper is the wrong fight.** Brittle per-store HTML parsing is why generalizing stalled. Better architecture: fetch the page, hand the raw content to an AI extraction step (Claude API or your local Qwen2.5-Coder via Ollama) that returns structured dimensions/model data as JSON, validated against a schema. One extraction prompt replaces N fragile parsers, and new stores work with zero code changes.
- **PyRevit constraints need stating upfront.** IronPython vs. CPython, no arbitrary pip installs inside Revit — the extension should shell out to a standard Python environment for scraping/extraction and keep only family generation inside Revit. That split is the design decision that unblocks the "self-contained extension" goal.
**Handoff:** Start a fresh session; provide the Gemini chat summaries you already compiled plus the current code. First deliverable is an architecture decision doc (extraction approach, PyRevit/Python split), *then* code — phase-gated, well-commented, novice walkthroughs.

### #3 — Raspberry Pi Consolidation (Home Assistant + alarm, one device)
**Status:** Two overlapping plans exist. The alarm implementation plan (finished yesterday) is ready for handoff and assumes cron + pychromecast on a Pi. Separately, the Home Assistant hardware research recommended a Pi 4 (2GB) — and an earlier session already noted HA's native Google Cast integration could absorb the alarm job.
**Why it's third:** Not a redo — a merge decision that should happen *before* you buy hardware or run the alarm plan. One Pi, one project, no duplicated infrastructure.
**The decision to make:** Either (a) run the alarm plan as written now (standalone scripts, simplest path, fully planned) and migrate into HA later, or (b) install Home Assistant OS first and build the alarm as an HA automation from day one — slightly more upfront learning, but one system to maintain and the calendar/cast pieces are native integrations. My lean is (b) if you're committed to HA anyway; (a) if you want the alarm working this week. Note: HA OS wants a 32GB card and the 2GB Pi 4 is adequate but snug with logging — worth a quick re-check at purchase time.
**Handoff:** Give the next model both docs (alarm plan + Pi hardware summary) and this decision framing. Output: one merged build plan.

### #4 — Shui Pái Structural Refactor + Feature Pass
**Status:** Working, deployed, actively played. Grew V1→V5+ through full-file rewrites in chat.
**Why a re-pass helps:** The full-rewrite iteration pattern is now the bottleneck — every change risks regressing something, and the single file has grown past what chat-based rewrites handle safely. Moving iteration into Claude Code (repo already the pattern from weather-dashboard) gets you surgical diffs, version history, and safe experimentation. A light internal reorganization (separating game state, rendering, and input handling within the single file — deployment stays single-file) makes future features much cheaper.
**Good candidates once refactored:** daily-seed challenge mode, stats screen, haptics via the Vibration API, additional themes.

### #5 — Vortex Polish
**Status:** Working, companion files done. Lowest urgency — same refactor-in-Claude-Code logic as Shui Pái applies if you return to it, but nothing is blocking or degrading. Park it.

### Deliberately not ranked
- **Google Home alarm:** already packaged as of yesterday — it's absorbed into #3 rather than redone.
- **Linux dual-boot (Zorin) & Ollama setup:** stalled on *your* hands-on steps (flash the ISO; install Ollama), not on planning quality. No Claude re-pass needed — just checklist execution when you have an evening. Note Ollama becomes load-bearing if you pick the local-extraction route in #2.

---

## Suggested Sequence

1. **Build Skill 1 (project-handoff) first** — one session, and it improves every handoff below.
2. **Weather dashboard** — hand off with the consolidation instructions above; Sonnet in Claude Code is well-suited.
3. **Pi decision + merged plan** — short session, do before buying hardware.
4. **Revit family generator** — architecture doc first, then build; worth Opus for the architecture session.
5. **Skills 2–4** as their trigger situations next arise (build the spec skill the next time a spec question comes up — real input beats hypothetical).
6. **Shui Pái refactor** — whenever you next want a feature; do the refactor as step one of that session.

---

*A note on sourcing: everything above is drawn from our conversation history via search. Recency bias applies — if a project I've marked stalled has moved since our last chat about it, trust your current state over this doc.*
