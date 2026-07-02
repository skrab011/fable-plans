# CLAUDE.md — Kitchen Equipment BIM Automation Tool

## What this project is
A pyRevit extension for Stais Architecture & Interiors that turns kitchen-equipment product URLs into parametric Revit families. An architect clicks a button in Revit, pastes product URLs, and receives flexed `.rfa` families (correct Width/Depth/Height, manufacturer/model/URL metadata) saved to a project folder on the office server. Purpose: high-level early-stage space planning on the architectural side — **not MEP** (no voltage/wattage data, deliberately).

## Who you're working with
Jacob is an expert architect and a coding novice. Explain what code does in plain language as you write it — logic, structure, and meaningful decisions — alongside comments, not instead of them. Ask clarifying questions before significant output. Push back with reasoning if an approach looks wrong; don't comply silently. Flag uncertainty directly. Use Plan Mode before significant builds. Record corrections and decisions into `implementation-plan.md` in the same session — it is the project memory.

## Architecture (three files + a bridge)
```
KitchenEquipment.pushbutton/
  script.py    ← pyRevit button (IronPython, runs inside Revit)
  scraper.py   ← web scraper (CPython + Playwright, runs OUTSIDE Revit)
  config.py    ← shared settings, imported by both
```
**The critical constraint:** Revit/pyRevit runs IronPython; Playwright requires CPython. So `script.py` launches `scraper.py` as a **subprocess** in the system CPython, passing URLs via stdin. The two halves communicate through one artifact: a CSV at `TEMP_CSV_PATH` (`%LOCALAPPDATA%\Temp\kitchen_equipment_scrape.csv`). Never try to import Playwright inside Revit; never try to touch the Revit API from scraper.py. The CSV is the contract.

## The CSV contract
Column order is defined once, in `config.py :: CSV_COLUMNS`:
`Equipment Type, Manufacturer, Width, Depth, Height, Finish, Model, URL, Notes`
Both scripts import it from config. Never reorder without updating both sides. Dimensions are written as decimal **inches** to 4 places (`"0.0000"` when parsing failed); script.py converts to decimal feet at the point of Revit injection.

## Scraper internals (already built — understand before changing)
- **`SITE_ROUTERS`** — a domain→function dictionary. Each supported store has one self-contained scrape function taking `(page, url)` and returning a dict keyed by `CSV_COLUMNS`. This is the adapter pattern: adding a store = writing one function + one dictionary entry. Currently supported: `webstaurantstore.com`, `restaurantsupply.com`.
- **`xray_find_spec(page, label)`** — generic fallback that hunts a labeled spec across four common HTML layouts (dt/dd, th/td, td/td, data-attributes). Site functions try their specific selectors first, then fall back to this. When a store's dimensions come back 0.0, the fix is usually a new strategy here or a new label mapping in the site function (e.g., "Overall Width" → Width).
- **The Machete engine (`machete_clean`)** — strips model numbers, electrical specs, dimensions, finish keywords, and SEO noise from product titles to produce a clean Equipment Type (which becomes the family name).
- **`parse_dimension`** — converts messy strings (`2'-6 1/2"`, `30.5"`) to decimal inches; returns 0.0 on failure, which the family generator must skip safely rather than write.
- **CAPTCHA handling:** the browser launches `headless=False` deliberately so a human can click CAPTCHAs; the session cookie usually holds afterward. Do not "optimize" this to headless.

## Revit-side conventions
- Two seed templates (FlexBox `.rft`), routed by `PLUMBING_KEYWORDS` in config: anything sink/wash/drain-like → `FlexBox_Plumbing.rft`, everything else → `FlexBox_Specialty.rft`.
- Family parameters `Width/Depth/Height/Finish/Notes` must match template parameter names exactly (case-sensitive; names live in config). `Manufacturer/Model/URL` map to Revit built-ins (`ALL_MODEL_MANUFACTURER`, `ALL_MODEL_MODEL`, `ALL_MODEL_URL`).
- Output `.rfa` files save to a user-picked project folder on the office server (folder picker at runtime — paths vary by project, so never hardcode).

## Standing decisions (do not relitigate without new information)
- **All-in-one pyRevit extension** over the earlier standalone GUI — chosen for simplicity and office shareability.
- **AI-based extraction is deferred, not rejected.** SAI has no AI-usage policy yet, so no Claude API calls and no local-model dependencies in this tool for now. The adapter pattern is the insurance: when policy lands, an AI extractor becomes one more adapter, with zero rework elsewhere. Revisit only when Jacob says the policy question is settled.
- **Architectural scope only:** no voltage/wattage/electrical fields.

## Working style for this repo
Well-commented code with the existing banner-comment style (see any current file). Test the scraper standalone (`python scraper.py`, manual URL paste) before testing inside Revit — the two failure domains should never be debugged simultaneously. Deliverables and corrections get recorded in `implementation-plan.md`.
