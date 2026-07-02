---
name: project-handoff
description: Package research, planning, or design work from a conversation into a handoff document that a future Claude session (Claude Code, Opus, or Sonnet) can execute from without re-deriving context. Proactively suggest using this skill whenever a planning or research session reaches a natural stopping point, whenever Jacob says work will continue "later," "in Claude Code," "on another machine," or with another model, whenever a multi-session project pivots direction, or whenever a long discussion has produced decisions that would otherwise live only in the chat thread. Also use when Jacob explicitly asks for a "handoff doc," "implementation plan," "CLAUDE.md," or to "package this up."
---

# Project Handoff

This skill packages the current conversation's decisions, research, and plans into documents a fresh Claude session can execute from. It exists because context dies with the thread — the handoff doc is how Jacob's best thinking survives the session boundary.

## Operating mode

**Trigger: suggest proactively, once.** When the conversation hits a natural handoff moment (see description), offer to package it — one short suggestion, then drop it if declined. Never nag.

**Gathering: draft first, interview gaps only.** Do NOT open with a questionnaire. Draft the full document from conversation history, memory, and any attached files. Then present it alongside a short numbered list of *only* the genuine gaps — things the thread never established. Never invent an unstated decision to fill a hole; mark it `OPEN` in the doc and put it on the gap list instead.

## Choosing the output format (adaptive default)

- **Code project** (something with a repo, or that will have one) → **two files:**
  - `CLAUDE.md` — persistent context the implementing session auto-loads: what the project is, architecture, conventions, standing preferences. Stable; changes rarely.
  - `implementation-plan.md` — the working document: phases, acceptance tests, open questions, dead-ends. Evolves; gets updated as work proceeds.
  - Keep them non-overlapping. CLAUDE.md says *what this is and how we work*; the plan says *what to do next and how we'll know it worked.*
- **Everything else** (research → action, purchase decision, process/hardware setup, decision memo) → **one handoff doc** containing all sections below.
- If the project already has a CLAUDE.md, do not regenerate it — produce a redline-style list of additions/changes instead.

## Required sections

Every handoff doc (or plan file, for the two-file format) contains these, in this order. Omit a section only if it's genuinely empty — and say so rather than silently dropping it.

1. **Goal** — one paragraph. What done looks like, for whom, and why it matters. No feature lists here.
2. **Current status** — what exists right now: files, repos, working pieces, hardware on hand, accounts/keys created. The implementing session should never have to guess what's already real.
3. **Locked decisions (with rationale)** — every decision made in the thread, each with its *why*. The rationale is what prevents a future session from helpfully "improving" a settled choice. If a decision reversed an earlier one, note both.
4. **Dead-ends table** — three columns: *What was tried / Why it failed / Don't retry unless…* This is often the highest-value section; it's the only record that saves a future session from repeating expensive mistakes.
5. **Open questions** — numbered, addressed to the implementing session, phrased as interview questions to ask Jacob before building. Include Jacob-homework items (e.g., "get the company's observed holiday list") distinctly marked.
6. **Phased plan with acceptance tests** — see phase rules below.
7. **Failure-mode defaults** — for anything that can fail at runtime (fetches, feeds, hardware), state explicitly whether it fails loud (do the safe thing anyway, alert) or fails silent (skip, log). Never leave this implicit; choose based on which failure hurts more, and record the reasoning.
8. **Resources** — repo URLs, file locations, relevant APIs, credential handling notes (never paste actual tokens/keys into the doc — describe where they live and how to create them just-in-time).

## Phase rules

- **Phase 1 is always the riskiest unknown**, not the logical first step. If an undocumented feed, an unproven hardware interaction, or a fragile dependency could sink the plan, prove it works in the first hour — before building anything on top of it. Name the risk explicitly in the phase title.
- Every phase ends with **acceptance tests**: concrete, observable checks Jacob can perform ("the Mini plays the MP3 at 35% volume with no voice announcement"), not abstractions ("casting works").
- Phases should be independently completable in a session. If a phase can't be tested without the next one, merge them.
- Prefer fewer, meatier phases over many thin ones.

## Standing instructions to embed in every handoff

Include a short "Working with Jacob" block in each doc (in CLAUDE.md for the two-file format) containing:

- **Interview first.** Ask the open questions before writing code or making purchases.
- **Explain in plain language.** Jacob is an expert architect and a coding novice. Walk through what code does and why, alongside comments — not instead of them.
- **Use Plan Mode** (or equivalent up-front planning) before any significant build step.
- **Push back, don't comply silently.** If the plan contains a flaw, say so with reasoning before executing.
- **Record corrections back into the doc.** When Jacob corrects course mid-build, update the implementation plan (decisions, dead-ends) in the same session. The doc is the project memory.
- **Flag uncertainty directly** rather than delivering confident guesses.

## Quality bar (self-check before delivering)

- Could a fresh session with zero chat history execute from this doc alone? If any step requires "remembering the conversation," the doc has failed.
- Does every locked decision carry its rationale?
- Is the riskiest unknown Phase 1?
- Are all acceptance tests observable by a human without reading code?
- Are the `OPEN` items short and genuinely unanswerable from the thread — not laziness?
- Is it as short as it can be while passing the checks above? Cut anything a competent session would figure out on its own. The dead-ends table and rationales are the last things to cut, ever.

## Delivery

- Save as markdown file(s) and present them — never paste the doc as a chat message.
- **Storage:** the doc's home is the project's Git repo or project folder (commit it there if the session has repo access; otherwise tell Jacob where it belongs). Then always remind Jacob to also save a copy to Obsidian for offline storage — every delivery, no exceptions.
- After delivering, give a 2–4 sentence summary flagging: the riskiest phase, any Jacob-homework items, and the gap questions awaiting answers.
- Suggest which model/surface fits the execution work (Claude Code for repo work; a chat session for research-to-decision tasks) — one sentence, not a pitch.

## Compact skeleton (single-doc format)

```markdown
# [Project] — Handoff & Implementation Plan
Prepared: [date] · Source: [chat thread topic] · Executor: [Claude Code / chat session]

## Goal
## Current status
## Locked decisions
| Decision | Rationale |
## Dead-ends
| Tried | Why it failed | Don't retry unless |
## Open questions (ask Jacob before building)
1. …
## Phased plan
### Phase 1 — [riskiest unknown, named]
Steps… **Acceptance tests:** …
### Phase 2 — …
## Failure-mode defaults
## Resources
## Working with Jacob
```

For the two-file format, move *Goal / architecture summary / conventions / Working with Jacob / Resources* into `CLAUDE.md` and keep everything phase- and decision-related in `implementation-plan.md`.
