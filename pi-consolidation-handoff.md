# Raspberry Pi Consolidation — Handoff & Implementation Plan
Prepared: July 1, 2026 · Source: Home Assistant hardware research + Google Home alarm implementation plan (June 2026 threads) · Executor: chat session (Opus/Sonnet) for the decision phase; the build itself is hands-on with Claude guiding

## Goal
One Raspberry Pi serving two jobs that were planned separately: (1) a Home Assistant server controlling ~a dozen smart devices with data logging and automations, and (2) the workday wake-up alarm that casts a local MP3 to the Google Home Mini on workday mornings only. Done looks like: a single device, a single maintenance surface, no duplicated infrastructure, and no cloud services beyond a healthchecks.io dead-man ping.

## Current status
- **Alarm implementation plan: complete** (June 30 session). Standalone-script architecture: cron + Python on the Pi, calendar via Google Calendar's secret iCal URL, casting via pychromecast to the Mini at a fixed IP. Jacob has this doc — the executing session should ask for it.
- **Pi hardware research: complete.** Pi 4 (2GB) recommended over Pi Zero 2 W; component list and pricing established (~$80–120 all-in: 32GB microSD, 5V/3A USB-C PSU, case, optional heatsink). `OPEN` — hardware not confirmed purchased.
- **Network:** WiFi-only for now; possible wired Ethernet if Jacob moves.
- **Known HA facts from research:** HA can back up its database to Google Drive via automation; HA has a native Google Cast integration; HA has native calendar integrations.
- **Target speaker:** Google Home Mini, 1st gen, fixed IP planned.

## Locked decisions
| Decision | Rationale |
|---|---|
| Raspberry Pi 4 over Pi Zero 2 W | HA + logging + automations workload would make the Zero 2 W sluggish |
| No cloud compute layer anywhere | The Pi does both "decide" and "play" halves; eliminates the GCP billing blocker, the service-account setup, and the cloud→Pi handoff seam. Local cron/automations also make Mountain Time DST handling free |
| Calendar access via secret iCal URL | No Calendar API, no GCP project, no JSON keys — a private URL returning the calendar as a plain file |
| Alarm behavior spec | Workday check = weekday AND not a company-observed holiday AND no "Your time off" PTO event; casts a locally hosted MP3 every 5 min across 5:30–6:05 and 6:30–7:05; 35% volume set silently in code; no voice announcements, no mute trick |
| Alarm fails loud | On any calendar-fetch failure the alarm fires anyway — a spurious wake-up on a PTO day is annoying; oversleeping on a workday is a real problem |
| healthchecks.io dead-man ping | The one permitted cloud service, so Jacob knows if the Pi goes down |
| "Decide" and "play" stay separate concerns | Even on one device, keep them as separate functions/automations — the mental model that survived the architecture change |

## Dead-ends
| Tried | Why it failed | Don't retry unless |
|---|---|---|
| Two-layer cloud plan (Cloud Scheduler + GCP functions → Pi) | Blocked on GCP billing setup; entire layer unnecessary once a Pi exists | The Pi approach fails outright |
| Google Calendar API + service account | Replaced by the secret iCal URL — same data, none of the setup | Google deprecates secret iCal URLs |
| Pi Zero 2 W as the server | Underpowered for HA + logging workload | Scope shrinks to alarm-only forever |
| Voice announcements / mute-trick volume control | Rejected in spec — volume must be set silently in code | Never |
| Pi as laptop replacement | Dropped once the Precision 5510 was repaired; unrelated to this project | — |

## Open questions (ask Jacob before building)
1. **The core decision — Option A or B?**
   - **A: Alarm first, standalone.** Execute the existing alarm plan as written (cron + scripts). Install HA later; migrate the alarm into HA whenever. *Wins:* alarm working within days; the plan is already fully phased and tested on paper. *Costs:* a migration later, and two systems briefly coexisting.
   - **B: Home Assistant first.** Install HA OS, then build the alarm as an HA automation using the native calendar + Google Cast integrations. *Wins:* one system from day one; device control, logging, and the alarm share the same backup/monitoring story. *Costs:* HA learning curve before the alarm exists; the existing alarm plan gets translated rather than executed.
   - Relevant fact: HA's Cast integration uses pychromecast under the hood, so the casting risk is identical either way — Phase 1 below covers both routes.
2. **Hardware purchased yet?** If not: confirm 2GB vs 4GB before ordering. HA OS runs on 2GB but is snug once device logging accumulates; the price delta is small. (`OPEN`)
3. **Jacob homework:** the company's *observed* holiday list — July 4, 2026 falls on a Saturday, so SAI presumably observes Friday July 3. The check needs observed dates, not nominal ones. (Carried from the alarm plan.)
4. **Jacob homework:** pick/prepare the alarm MP3 file.
5. **What are the ~12 smart devices** (brands/protocols)? Determines HA integrations needed and whether a Zigbee/Z-Wave USB dongle joins the shopping list, or if everything is WiFi. (`OPEN` — never enumerated in any thread.)
6. **Can the router assign a DHCP reservation** for the Mini? The fixed-IP requirement depends on it. (`OPEN`)

## Phased plan
### Phase 0 — Decision gate
Interview Jacob on Open Questions 1–2. No purchases or builds until both are answered.
**Acceptance test:** Option chosen and recorded here with rationale; hardware order placed or confirmed on hand.

### Phase 1 — Prove the cast (riskiest unknown)
Flash the Pi (HA OS if Option B; Raspberry Pi OS Lite if A), boot, join WiFi, set the Mini's DHCP reservation. Then a bare-bones test: cast a local MP3 to the 1st-gen Mini at 35% volume with no voice announcement — via an HA service call (B) or a minimal pychromecast script (A). This is first because pychromecast behavior against this specific 1st-gen Mini and network is the one thing that can sink the whole plan, and it's provable in the first hour.
**Acceptance tests:** the Mini plays the chosen MP3; volume lands at 35% without any audible announcement or chime; the cast works twice in a row after the Pi reboots.

### Phase 2 — Workday decision logic
Connect the secret iCal URL. Implement the workday check (weekday / observed holidays / "Your time off" events) as its own testable function or template.
**Acceptance tests:** a normal Tuesday returns "fire"; a date on the observed-holiday list returns "skip"; a day with a "Your time off" event returns "skip"; unplugging the network (fetch failure) returns "fire" — the fail-loud default, observably.

### Phase 3 — Schedule + monitoring
Wire the two morning windows (every 5 min, 5:30–6:05 and 6:30–7:05) to the Phase 1 cast and Phase 2 check. Add the healthchecks.io ping.
**Acceptance tests:** a compressed dry run (windows temporarily set to the next 10 minutes) fires on schedule; healthchecks.io shows green, and shows the alert firing when the Pi is unplugged for the ping interval.

### Phase 4 — Home Assistant buildout (immediate under B; deferred under A)
Onboard the ~12 devices, set up data logging, and configure the Google Drive backup automation. Under Option A, this phase also includes migrating the alarm into HA and retiring the standalone scripts — one system at the end either way.
**Acceptance tests:** every device controllable from the HA dashboard; a backup file appears in Google Drive on schedule; (if migrating) the standalone cron jobs are removed and the alarm still passes the Phase 3 dry run.

## Failure-mode defaults
- **Alarm path: fails loud.** Any calendar or fetch failure → alarm fires anyway. Deliberate; do not "improve" this.
- **HA logging/automations: fail silent-with-log.** A missed sensor reading shouldn't page anyone.
- **Device liveness: healthchecks.io** is the single external alert channel.

## Resources
- Existing alarm implementation plan (markdown; Jacob has it — request it at session start)
- Raspberry Pi Imager for flashing; HA OS image from home-assistant.io (Option B)
- Secret iCal URL: Google Calendar → Settings → "Secret address in iCal format" — treat it like a password; never paste it into docs or commits
- healthchecks.io free tier
- No API keys required for the alarm path; create any HA-related credentials just-in-time

## Working with Jacob
- Interview first — resolve the Open Questions above before building or buying.
- Jacob is an expert architect and a coding novice: explain what code and configs do in plain language as you go, alongside comments.
- Plan before significant build steps; push back with reasoning if something here looks wrong rather than complying silently.
- Record corrections and decisions back into this document in the same session — this doc is the project memory.
- Flag uncertainty directly.
