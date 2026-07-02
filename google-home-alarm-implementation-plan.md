# Implementation Plan: PTO-Aware Morning Alarm — Raspberry Pi + Google Home Mini

**Prepared for:** Claude Opus 4.8 / Sonnet (implementing session)
**Prepared by:** Claude Fable 5, for Jacob
**Date:** July 1, 2026
**Supersedes:** "Project Handoff: PTO-Aware Conditional Morning Alarm" (Opus 4.8, July 1, 2026)

**Rev log:**
- **Rev 2 — July 2, 2026:** Section 8 interview completed with Jacob; answers locked in throughout (see Section 8). Volume changed 35% → **100%**. Wake audio: Claude sources a royalty-free alarm tone during the build (the Google Home's native alarm tone can't be cast — it has no media URL). Holidays: SAI observes weekend holidays on the nearest weekday. Second window re-runs the calendar check. Phase 0 network/healthchecks prep not yet done.

---

## 0. Read This First — Context for the Implementing Claude

Jacob is a practicing architect. **Treat him as a coding novice**: explain what every script does in plain language alongside the code, walk through the logic and structure, and don't assume familiarity with syntax or patterns. He is comfortable with terminal use and step-by-step guidance, has used Claude Code, GitHub, and Markdown, and prefers peer-level directness. If something in this plan seems wrong or a better approach exists, **push back with reasoning** — don't silently comply.

Work **one phase at a time**. Complete and verify each phase's acceptance test before moving to the next. Before writing any code, run the interview in Section 8.

---

## 1. Goal (Functional Spec — Final)

A Raspberry Pi Zero 2 W on Jacob's home network wakes him via his **Google Home Mini (1st gen)** on workday mornings only:

- **Skip weekends** (Sat/Sun)
- **Skip six holidays:** New Year's Day, Memorial Day, Independence Day, Labor Day, Thanksgiving, Christmas
- **Skip PTO days:** Justworks PTO syncs to Google Calendar as all-day events titled **"Your time off"** (confirmed working)
- **On a valid workday:** set Mini volume to **100%** silently, then play a wake sound **every 5 minutes** across two windows: **5:30–6:05 AM** and **6:30–7:05 AM** Mountain Time (8 plays per window, 16 total)

## 2. Architecture (Decided — Do Not Relitigate)

**Single device does everything.** The Pi runs both the decision logic and the Cast playback. There is no cloud layer, no Google Cloud project, no service account, no billing. These were part of an earlier two-layer design and have been **deliberately removed**.

| Concern | Mechanism |
|---|---|
| Scheduling | `cron` on the Pi, timezone set to `America/Denver` (DST handled automatically) |
| Calendar access | Google Calendar **secret iCal URL** ("Secret address in iCal format" in calendar settings) fetched with `requests`, parsed with the `icalendar` library. No OAuth, no API keys. |
| Decision timing | **Morning-of check**: the script runs at 5:25 AM, decides whether *today* is a workday, and either proceeds or exits. No night-before job, no state file. (Improvement over original spec: catches PTO added the night before.) |
| Playback | `pychromecast`, connecting to the Mini **by fixed IP** (DHCP reservation), not mDNS discovery — this avoids the most common Cast reliability failure. |
| Volume | `cast.set_volume(1.0)` — silent, set in code. Define it as a named constant (e.g. `ALARM_VOLUME = 1.0`) with a plain-language comment telling Jacob how to lower it (0.0–1.0 scale). **The original "mute trick" (volume 0 → 16 voice-created alarms → restore) is obsolete and must not be implemented.** Direct casting never speaks confirmations. |
| Wake audio | An MP3 served from the Pi itself via a small local HTTP server (systemd service), so playback has zero external dependencies and personal audio is allowed (Voice Match restrictions don't apply to direct Cast). |
| Monitoring | Free healthchecks.io check pinged on every successful morning run (including "skipped — PTO/weekend/holiday" runs). Jacob gets an email if the Pi dies. |

### Hard constraints (carried forward — do not re-explore)
- All-Apple personal hardware; iOS sandboxing blocks catt/pychromecast on iPhone/iPad (validated by testing)
- Google Home has **no** native calendar-conditional triggering (re-confirmed mid-2026)
- The Home Automation API is an on-device mobile SDK, not server-callable
- IFTTT / SmartThings-command / Make.com chains to Google Assistant are dead
- Voice Match blocks personal content in household script automations (irrelevant here, since we cast directly)

## 3. Hardware & Prerequisites (Phase 0)

**To purchase:**
- Raspberry Pi **Zero 2 W** (not the original Zero — needs WiFi anyway, and the Zero 2's quad-core handles modern Python far better for ~$5 more)
- microSD card, 16–32 GB, name brand
- Official 5V power supply (micro-USB)
- No HDMI cable, keyboard, or case required — this is a **headless** build (SSH only)

**To set up before the Pi arrives** *(status as of July 2, 2026: none of these done yet)*:
1. Router: create **DHCP reservations** (fixed IPs) for the Google Home Mini and, later, the Pi. Record both IPs. ⏳ Not yet done — required before Phase 2.
2. Google Calendar (web): Settings → the calendar receiving Justworks PTO → "Integrate calendar" → copy the **Secret address in iCal format**. Treat it like a password.
3. Create a free **healthchecks.io** account; create one check with a daily schedule and a generous grace window; copy the ping URL. ⏳ Not yet done — needed by Phase 6; the ping URL lives only on the Pi, never in this repo.
4. Wake-sound MP3: **resolved** — the implementing Claude sources a royalty-free alarm tone during the build; Jacob approves it in Phase 2 testing. Same sound for both windows. (The Mini's native alarm tone is not an option: it exists only inside the Assistant's alarm feature and has no media URL to cast.)

## 4. Implementation Phases

### Phase 1 — Pi bring-up (headless)
- Flash **Raspberry Pi OS Lite (64-bit)** with Raspberry Pi Imager; use the Imager's settings to pre-configure hostname, WiFi credentials, SSH, and username *before* first boot.
- SSH in from Jacob's machine; `sudo apt update && sudo apt full-upgrade`.
- Set timezone: `sudo raspi-config` → `America/Denver`. Verify with `date`.
- Create project directory (`~/alarm/`), a Python **venv** (required — modern Raspberry Pi OS blocks system-wide `pip` installs per PEP 668), and install: `pychromecast`, `requests`, `icalendar`.
- **Acceptance test:** SSH works; `date` shows correct MT time; `python -c "import pychromecast"` succeeds inside the venv.

### Phase 2 — Cast proof-of-concept (do this before any calendar work)
- Short script: connect to the Mini by its fixed IP, print device status, `set_volume(0.35)`, play a test MP3 from a public URL, stop.
- This de-risks the hardest unknown (1st-gen Mini + pychromecast on this network) first.
- **Acceptance test:** Mini plays the test audio at 35% with no spoken announcements.

### Phase 3 — Local audio hosting
- Place the wake MP3 in `~/alarm/audio/`.
- Small HTTP server on the Pi (Python `http.server` wrapped in a **systemd service** so it survives reboots), serving that directory on a fixed port.
- Update the Phase 2 script to cast `http://<pi-ip>:<port>/wake.mp3`.
- **Acceptance test:** reboot the Pi; the audio URL loads in a browser on Jacob's phone; the cast test still works.

### Phase 4 — Decision logic
- `decide.py` (or a function within the main script) answering one question: **"Is today a wake day?"**
  1. **Weekday check** — `datetime.now()` local; Sat/Sun → no.
  2. **Holiday check** — the six holidays. **Confirmed (interview Q2): SAI observes weekend holidays on the nearest weekday** (e.g., Friday July 3 for July 4, 2026). Use the Python `holidays` library filtered to the six, with observed-date handling enabled.
  3. **PTO check** — fetch the secret iCal URL, parse with `icalendar`, and check whether any all-day event titled "Your time off" overlaps today. Match generously (case-insensitive, substring) and log what was found. Handle multi-day PTO events (date ranges), noting iCal all-day events use an **exclusive** end date.
- Robustness: if the calendar fetch fails (network blip), **default to sounding the alarm** and log the error — a spurious wake-up beats a missed one.
- **Acceptance test:** run manually with today's date and with mocked dates (a Saturday, July 3 2026, a fake PTO day, a normal Tuesday); each returns the correct decision with a clear log line explaining why.

### Phase 5 — The alarm sequence + scheduling
- `alarm.py`: runs the decision; if "no," logs the reason, pings healthchecks, exits. If "yes": connects to the Mini, sets volume to 0.35, then plays the MP3 at each interval. Structure as **one process per window** that sleeps between plays (8 plays, 5 minutes apart), reconnecting to the Mini per play rather than holding one socket open for 35 minutes (Cast sockets drop; fresh connections are more reliable).
- Two cron entries: `25 5 * * 1-5` (decision + window 1 at 5:30) and `30 6 * * 1-5` (window 2 — re-run the decision cheaply or trust window 1's log; recommend re-running, it costs nothing).
- Log everything to `~/alarm/alarm.log` with timestamps.
- **Acceptance test:** temporarily set cron to a daytime hour; observe a full 2-play shortened sequence end-to-end, then restore real times.

### Phase 6 — Reliability hardening
- healthchecks.io ping (`requests.get(ping_url)`) at the end of every run, success or skip. Fail-ping (`/fail` endpoint) on exceptions.
- Confirm WiFi auto-reconnect (default on RPi OS) and consider a nightly reboot cron (`@daily`? No — pick 3 AM, well clear of alarm windows) if long-uptime flakiness appears. Don't add it preemptively.
- Full dry-run week: add a fake "Your time off" event mid-week and verify the skip; verify a normal day fires.
- **Acceptance test:** pull the Pi's power for a day → Jacob receives a healthchecks alert email.

## 5. Testing Matrix (run before trusting it)

| Scenario | Expected |
|---|---|
| Normal Tuesday | Full 16-play sequence at 35% |
| Saturday | Skip, logged "weekend" |
| Fri Jul 3, 2026 (observed July 4th) | Skip, logged "holiday" — confirmed: SAI observes it |
| All-day "Your time off" event today | Skip, logged "PTO" |
| Multi-day PTO spanning today | Skip |
| Calendar URL unreachable | **Alarm fires anyway**, error logged |
| Pi powered off | healthchecks email within grace window |

## 6. Explicitly Out of Scope
- The HomePod Mini / Apple-native track (separate project)
- The future Raspberry Pi 4 + Home Assistant build (this script's logic can migrate there later; don't design for it now)
- The native `OkGoogle` public-media fallback — **rejected**: a fallback that can't read the calendar fires on PTO days, failing in exactly the way this project exists to prevent

## 7. Documented Fallback: Cloud-Only "Virtual Switch" Chain (only if the Pi path is abandoned)

For the record, one no-hardware architecture exists: Google Apps Script (free, built-in Calendar access, nightly trigger) flips a SmartThings **virtual switch** via the SmartThings REST API; the switch appears in Google Home via the standard SmartThings link; a Google Home script-editor automation fires weekday mornings **conditioned on that switch**, using `OkGoogle` actions for volume and media. Limitations: public media only (Voice Match), best-effort reliability, three-cloud dependency chain, and the linked-switch-as-script-condition combo is **unverified end-to-end** — requires a proof-of-concept before building on it. This is a fallback, not the plan.

## 8. Interview Jacob Before Writing Code — ✅ COMPLETED July 2, 2026

Answers are locked in; the implementing session does not need to re-ask these.

1. **Wake audio:** Jacob asked for the Google Home's default alarm tone — not possible (it has no castable media URL). **Decision: the implementing Claude finds a suitable royalty-free alarm tone during the build**; Jacob approves it in Phase 2 testing. Same sound for both windows.
2. **Holiday observed dates:** **Yes — SAI observes weekend holidays on the nearest weekday.** Use observed-date handling (e.g., skip Fri Jul 3, 2026).
3. **Mini's fixed IP / Pi hostname:** **Not set up yet.** DHCP reservations still to do (Phase 0). Collect the actual IPs at the start of the build session, before Phase 2.
4. **Volume:** **100%** (`1.0`), up from the specced 35% — Jacob currently uses 100%. Put it in a named constant with a plain-language code comment explaining how to adjust it.
5. **Second window decision re-check:** **Re-run the calendar check at 6:30** (as recommended). Each cron job is self-contained; no shared state.
6. **healthchecks.io:** **Not created yet.** Needed by Phase 6; the ping URL goes only onto the Pi, never into this repo (no-secrets convention).

## 9. One-Paragraph Summary

A Raspberry Pi Zero 2 W runs the entire system: a 5:25 AM cron job fetches Jacob's Google Calendar via its secret iCal URL, decides whether today is a workday (weekday, not one of six company-observed holidays, no "Your time off" PTO event), and if so casts a locally-hosted MP3 (a royalty-free alarm tone sourced during the build) to the Google Home Mini (by fixed IP via pychromecast) every 5 minutes across 5:30–6:05 and 6:30–7:05 at 100% volume — set silently in code, no mute trick, no voice announcements. No cloud services except a healthchecks.io dead-man ping so Jacob knows if the Pi ever goes down. On any calendar-fetch failure, the alarm fires anyway. Build it phase by phase with the acceptance tests above, and explain every piece of code in plain language as you go.
